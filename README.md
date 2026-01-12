# **精简版：达梦数据库简单同步方案（Gradle + 存储过程）**

## **一、极简Gradle配置**

### **build.gradle.kts**
```kotlin
plugins {
    java
    id("org.springframework.boot") version "2.7.14"
    id("io.spring.dependency-management") version "1.1.0"
}

group = "com.dm.sync"
version = "1.0.0"
java.sourceCompatibility = JavaVersion.VERSION_11

repositories {
    mavenCentral()
    maven { url = uri("https://maven.aliyun.com/repository/public") }
}

dependencies {
    // 只需要最基础的依赖
    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
    
    // 达梦驱动（放在libs目录）
    implementation(files("libs/DmJdbcDriver18.jar"))
    
    // 连接池
    implementation("com.zaxxer:HikariCP:5.0.1")
    
    // 工具类
    implementation("org.apache.commons:commons-lang3:3.12.0")
    
    // JSON处理（简单配置需要）
    implementation("com.fasterxml.jackson.core:jackson-databind:2.15.2")
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
}

tasks.bootJar {
    archiveFileName.set("dm-sync-simple.jar")
}
```

## **二、极简数据库脚本**

### **1. 基础表结构（只有必须的）**
```sql
-- 1. 同步配置表（最简单的版本）
CREATE TABLE SYNC_CONFIG (
    TABLE_NAME VARCHAR(100) PRIMARY KEY,
    SOURCE_SCHEMA VARCHAR(50) DEFAULT 'SOURCE_SCHEMA',
    TARGET_SCHEMA VARCHAR(50) DEFAULT 'TARGET_SCHEMA',
    SYNC_MODE VARCHAR(20) DEFAULT 'AUTO', -- AUTO/TIME/KEY/FULL
    SYNC_COLUMN VARCHAR(50),
    PRIMARY_KEY VARCHAR(50),
    SYNC_INTERVAL_MINUTES INT DEFAULT 5,
    ENABLED CHAR(1) DEFAULT 'Y',
    LAST_SYNC_TIME TIMESTAMP
);

-- 2. 简单日志表（只记录最后状态）
CREATE TABLE SYNC_LAST_STATUS (
    TABLE_NAME VARCHAR(100) PRIMARY KEY,
    LAST_SUCCESS_TIME TIMESTAMP,
    LAST_ERROR_MSG VARCHAR(500),
    ROW_COUNT INT DEFAULT 0
);
```

### **2. 核心存储过程（一个通用过程搞定）**
```sql
-- 通用同步存储过程（处理所有情况）
CREATE OR REPLACE PROCEDURE PROC_SYNC_TABLE(
    IN_TABLE_NAME VARCHAR(100)
)
AS
    V_SOURCE_SCHEMA VARCHAR(50);
    V_TARGET_SCHEMA VARCHAR(50);
    V_SYNC_MODE VARCHAR(20);
    V_SYNC_COLUMN VARCHAR(50);
    V_PRIMARY_KEY VARCHAR(50);
    V_SQL VARCHAR(4000);
    V_ROW_COUNT INT := 0;
    V_HAS_TIME_COLUMN INT;
    V_HAS_INCREMENT_COLUMN INT;
BEGIN
    -- 获取配置
    SELECT SOURCE_SCHEMA, TARGET_SCHEMA, SYNC_MODE, SYNC_COLUMN, PRIMARY_KEY
    INTO V_SOURCE_SCHEMA, V_TARGET_SCHEMA, V_SYNC_MODE, V_SYNC_COLUMN, V_PRIMARY_KEY
    FROM SYNC_CONFIG
    WHERE TABLE_NAME = IN_TABLE_NAME AND ENABLED = 'Y';
    
    -- 如果没有配置同步字段，自动检测
    IF V_SYNC_COLUMN IS NULL THEN
        -- 检查是否有时间字段
        SELECT COUNT(*) INTO V_HAS_TIME_COLUMN
        FROM ALL_TAB_COLUMNS
        WHERE OWNER = V_SOURCE_SCHEMA
          AND TABLE_NAME = IN_TABLE_NAME
          AND (COLUMN_NAME LIKE '%TIME%' OR COLUMN_NAME LIKE '%DATE%' 
               OR DATA_TYPE LIKE '%TIMESTAMP%' OR DATA_TYPE LIKE '%DATE%');
        
        -- 检查是否有自增字段
        SELECT COUNT(*) INTO V_HAS_INCREMENT_COLUMN
        FROM ALL_TAB_COLUMNS
        WHERE OWNER = V_SOURCE_SCHEMA
          AND TABLE_NAME = IN_TABLE_NAME
          AND (COLUMN_NAME LIKE '%ID' OR COLUMN_DEFAULT LIKE '%IDENTITY%');
        
        -- 自动选择同步策略
        IF V_HAS_TIME_COLUMN > 0 THEN
            -- 使用第一个时间字段
            SELECT COLUMN_NAME INTO V_SYNC_COLUMN
            FROM ALL_TAB_COLUMNS
            WHERE OWNER = V_SOURCE_SCHEMA
              AND TABLE_NAME = IN_TABLE_NAME
              AND (COLUMN_NAME LIKE '%TIME%' OR COLUMN_NAME LIKE '%DATE%')
            ORDER BY COLUMN_ID
            LIMIT 1;
            
            V_SYNC_MODE := 'TIME';
            
        ELSIF V_HAS_INCREMENT_COLUMN > 0 THEN
            -- 使用自增字段
            SELECT COLUMN_NAME INTO V_SYNC_COLUMN
            FROM ALL_TAB_COLUMNS
            WHERE OWNER = V_SOURCE_SCHEMA
              AND TABLE_NAME = IN_TABLE_NAME
              AND (COLUMN_NAME LIKE '%ID' OR COLUMN_DEFAULT LIKE '%IDENTITY%')
            ORDER BY COLUMN_ID
            LIMIT 1;
            
            V_SYNC_MODE := 'KEY';
            
        ELSE
            V_SYNC_MODE := 'FULL';
        END IF;
        
        -- 更新配置
        UPDATE SYNC_CONFIG 
        SET SYNC_MODE = V_SYNC_MODE,
            SYNC_COLUMN = V_SYNC_COLUMN
        WHERE TABLE_NAME = IN_TABLE_NAME;
    END IF;
    
    -- 如果没有主键，尝试获取第一个字段作为主键
    IF V_PRIMARY_KEY IS NULL THEN
        SELECT COLUMN_NAME INTO V_PRIMARY_KEY
        FROM ALL_TAB_COLUMNS
        WHERE OWNER = V_SOURCE_SCHEMA
          AND TABLE_NAME = IN_TABLE_NAME
        ORDER BY COLUMN_ID
        LIMIT 1;
        
        UPDATE SYNC_CONFIG 
        SET PRIMARY_KEY = V_PRIMARY_KEY
        WHERE TABLE_NAME = IN_TABLE_NAME;
    END IF;
    
    -- 根据同步模式执行不同的同步逻辑
    CASE V_SYNC_MODE
        WHEN 'TIME' THEN
            -- 基于时间字段的增量同步
            V_SQL := '
                MERGE INTO ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || ' T
                USING (
                    SELECT * FROM ' || V_SOURCE_SCHEMA || '.' || IN_TABLE_NAME || ' S
                    WHERE S.' || V_SYNC_COLUMN || ' > COALESCE(
                        (SELECT MAX(' || V_SYNC_COLUMN || ') 
                         FROM ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || '),
                        TO_DATE(''1970-01-01'', ''YYYY-MM-DD'')
                    )
                ) S ON (T.' || V_PRIMARY_KEY || ' = S.' || V_PRIMARY_KEY || ')
                WHEN MATCHED THEN UPDATE SET ';
            
            -- 动态生成UPDATE SET子句
            DECLARE
                V_FIRST BOOLEAN := TRUE;
            BEGIN
                FOR COL IN (
                    SELECT COLUMN_NAME 
                    FROM ALL_TAB_COLUMNS 
                    WHERE OWNER = V_SOURCE_SCHEMA 
                      AND TABLE_NAME = IN_TABLE_NAME
                      AND COLUMN_NAME != V_PRIMARY_KEY
                ) LOOP
                    IF NOT V_FIRST THEN
                        V_SQL := V_SQL || ', ';
                    END IF;
                    V_SQL := V_SQL || 'T.' || COL.COLUMN_NAME || ' = S.' || COL.COLUMN_NAME;
                    V_FIRST := FALSE;
                END LOOP;
            END;
            
            V_SQL := V_SQL || '
                WHEN NOT MATCHED THEN INSERT VALUES (S.*)';
            
        WHEN 'KEY' THEN
            -- 基于键字段的增量同步
            V_SQL := '
                MERGE INTO ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || ' T
                USING (
                    SELECT * FROM ' || V_SOURCE_SCHEMA || '.' || IN_TABLE_NAME || ' S
                    WHERE S.' || V_SYNC_COLUMN || ' > COALESCE(
                        (SELECT MAX(' || V_SYNC_COLUMN || ') 
                         FROM ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || '),
                        0
                    )
                ) S ON (T.' || V_PRIMARY_KEY || ' = S.' || V_PRIMARY_KEY || ')
                WHEN MATCHED THEN UPDATE SET ';
            
            -- 动态生成UPDATE SET子句
            DECLARE
                V_FIRST BOOLEAN := TRUE;
            BEGIN
                FOR COL IN (
                    SELECT COLUMN_NAME 
                    FROM ALL_TAB_COLUMNS 
                    WHERE OWNER = V_SOURCE_SCHEMA 
                      AND TABLE_NAME = IN_TABLE_NAME
                      AND COLUMN_NAME != V_PRIMARY_KEY
                ) LOOP
                    IF NOT V_FIRST THEN
                        V_SQL := V_SQL || ', ';
                    END IF;
                    V_SQL := V_SQL || 'T.' || COL.COLUMN_NAME || ' = S.' || COL.COLUMN_NAME;
                    V_FIRST := FALSE;
                END LOOP;
            END;
            
            V_SQL := V_SQL || '
                WHEN NOT MATCHED THEN INSERT VALUES (S.*)';
            
        ELSE
            -- 全量同步（最简单的方式）
            V_SQL := '
                DELETE FROM ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || ';
                
                INSERT INTO ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || '
                SELECT * FROM ' || V_SOURCE_SCHEMA || '.' || IN_TABLE_NAME;
    END CASE;
    
    -- 执行同步
    IF V_SYNC_MODE = 'FULL' THEN
        -- 全量同步需要分步执行
        EXECUTE IMMEDIATE 'DELETE FROM ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME;
        EXECUTE IMMEDIATE 'INSERT INTO ' || V_TARGET_SCHEMA || '.' || IN_TABLE_NAME || 
                         ' SELECT * FROM ' || V_SOURCE_SCHEMA || '.' || IN_TABLE_NAME;
    ELSE
        EXECUTE IMMEDIATE V_SQL;
    END IF;
    
    V_ROW_COUNT := SQL%ROWCOUNT;
    
    -- 更新状态
    UPDATE SYNC_CONFIG 
    SET LAST_SYNC_TIME = SYSDATE
    WHERE TABLE_NAME = IN_TABLE_NAME;
    
    MERGE INTO SYNC_LAST_STATUS T
    USING (SELECT 
            IN_TABLE_NAME AS TABLE_NAME,
            SYSDATE AS LAST_SUCCESS_TIME,
            NULL AS LAST_ERROR_MSG,
            V_ROW_COUNT AS ROW_COUNT
          ) S
    ON (T.TABLE_NAME = S.TABLE_NAME)
    WHEN MATCHED THEN
        UPDATE SET 
            T.LAST_SUCCESS_TIME = S.LAST_SUCCESS_TIME,
            T.LAST_ERROR_MSG = S.LAST_ERROR_MSG,
            T.ROW_COUNT = S.ROW_COUNT
    WHEN NOT MATCHED THEN
        INSERT (TABLE_NAME, LAST_SUCCESS_TIME, LAST_ERROR_MSG, ROW_COUNT)
        VALUES (S.TABLE_NAME, S.LAST_SUCCESS_TIME, S.LAST_ERROR_MSG, S.ROW_COUNT);
    
    COMMIT;
    
EXCEPTION
    WHEN OTHERS THEN
        -- 记录错误
        MERGE INTO SYNC_LAST_STATUS T
        USING (SELECT 
                IN_TABLE_NAME AS TABLE_NAME,
                NULL AS LAST_SUCCESS_TIME,
                SQLERRM AS LAST_ERROR_MSG,
                0 AS ROW_COUNT
              ) S
        ON (T.TABLE_NAME = S.TABLE_NAME)
        WHEN MATCHED THEN
            UPDATE SET 
                T.LAST_SUCCESS_TIME = S.LAST_SUCCESS_TIME,
                T.LAST_ERROR_MSG = S.LAST_ERROR_MSG,
                T.ROW_COUNT = S.ROW_COUNT
        WHEN NOT MATCHED THEN
            INSERT (TABLE_NAME, LAST_SUCCESS_TIME, LAST_ERROR_MSG, ROW_COUNT)
            VALUES (S.TABLE_NAME, S.LAST_SUCCESS_TIME, S.LAST_ERROR_MSG, S.ROW_COUNT);
        
        ROLLBACK;
        RAISE;
END;
/

-- 批量同步所有表的存储过程
CREATE OR REPLACE PROCEDURE PROC_SYNC_ALL_TABLES
AS
    CURSOR C_TABLES IS
        SELECT TABLE_NAME 
        FROM SYNC_CONFIG 
        WHERE ENABLED = 'Y'
        ORDER BY TABLE_NAME;
BEGIN
    FOR TABLE_REC IN C_TABLES LOOP
        BEGIN
            PROC_SYNC_TABLE(TABLE_REC.TABLE_NAME);
            DBMS_LOCK.SLEEP(1); -- 表间等待1秒，避免压力过大
        EXCEPTION
            WHEN OTHERS THEN
                -- 继续同步其他表，不中断
                NULL;
        END;
    END LOOP;
END;
/

-- 创建定时作业（每5分钟同步一次）
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'JOB_SIMPLE_SYNC',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'PROC_SYNC_ALL_TABLES',
        start_date      => SYSDATE,
        repeat_interval => 'FREQ=MINUTELY;INTERVAL=5',
        enabled         => TRUE
    );
END;
/
```

### **3. 初始化配置脚本**
```sql
-- 初始化十几张表的配置
BEGIN
    -- 表1：用户表（有时间字段）
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, SYNC_COLUMN, PRIMARY_KEY) 
    VALUES ('USER_INFO', 'TIME', 'UPDATE_TIME', 'USER_ID');
    
    -- 表2：订单表（有时间字段）
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, SYNC_COLUMN, PRIMARY_KEY) 
    VALUES ('ORDER_INFO', 'TIME', 'CREATE_TIME', 'ORDER_ID');
    
    -- 表3：产品表（无时间字段，有自增ID）
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, SYNC_COLUMN, PRIMARY_KEY) 
    VALUES ('PRODUCT_INFO', 'KEY', 'PRODUCT_ID', 'PRODUCT_ID');
    
    -- 表4：部门表（无时间字段，无自增）
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, PRIMARY_KEY) 
    VALUES ('DEPARTMENT', 'FULL', 'DEPT_ID');
    
    -- 表5：角色表
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, PRIMARY_KEY) 
    VALUES ('ROLE_INFO', 'FULL', 'ROLE_ID');
    
    -- 表6：权限表
    INSERT INTO SYNC_CONFIG (TABLE_NAME, SYNC_MODE, PRIMARY_KEY) 
    VALUES ('PERMISSION', 'FULL', 'PERM_ID');
    
    -- 表7：日志表（有时间但可能不需要同步）
    INSERT INTO SYNC_CONFIG (TABLE_NAME, ENABLED, SYNC_MODE, SYNC_COLUMN, PRIMARY_KEY) 
    VALUES ('OPERATION_LOG', 'N', 'TIME', 'LOG_ID');
    
    -- 表8-15：其他业务表（使用AUTO自动检测）
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('CUSTOMER_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('SUPPLIER_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('WAREHOUSE_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('INVENTORY_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('INVOICE_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('PAYMENT_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('SHIPMENT_INFO');
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('CONTRACT_INFO');
    
    COMMIT;
END;
/
```

## **三、极简Java服务**

### **1. 配置文件 application.yml**
```yaml
spring:
  datasource:
    url: jdbc:dm://localhost:5236
    username: SYSDBA
    password: SYSDBA
    driver-class-name: dm.jdbc.driver.DmDriver
    hikari:
      maximum-pool-size: 5
      connection-timeout: 10000

sync:
  # 表配置（也可以在数据库配置）
  tables:
    - USER_INFO
    - ORDER_INFO
    - PRODUCT_INFO
    - DEPARTMENT
    - ROLE_INFO
    - PERMISSION
    - CUSTOMER_INFO
    - SUPPLIER_INFO
    - WAREHOUSE_INFO
    - INVENTORY_INFO
    - INVOICE_INFO
    - PAYMENT_INFO
    - SHIPMENT_INFO
    - CONTRACT_INFO
  
  # 同步间隔（秒）
  interval: 300
  
  # 是否使用数据库定时作业
  use-db-job: true
```

### **2. 数据源配置**
```java
package com.dm.sync.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;

@Configuration
public class DataSourceConfig {
    
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

### **3. 核心同步服务（不到100行）**
```java
package com.dm.sync.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import javax.annotation.PostConstruct;
import javax.sql.DataSource;
import java.util.*;

@Service
public class SimpleSyncService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Value("${sync.tables}")
    private List<String> tables;
    
    @Value("${sync.use-db-job:true}")
    private boolean useDbJob;
    
    private SimpleJdbcCall syncTableCall;
    private SimpleJdbcCall syncAllCall;
    
    @PostConstruct
    public void init() {
        DataSource dataSource = jdbcTemplate.getDataSource();
        if (dataSource != null) {
            syncTableCall = new SimpleJdbcCall(dataSource)
                .withProcedureName("PROC_SYNC_TABLE");
            
            syncAllCall = new SimpleJdbcCall(dataSource)
                .withProcedureName("PROC_SYNC_ALL_TABLES");
        }
    }
    
    /**
     * 同步单表
     */
    public boolean syncTable(String tableName) {
        try {
            Map<String, Object> params = new HashMap<>();
            params.put("IN_TABLE_NAME", tableName);
            
            syncTableCall.execute(params);
            System.out.println("同步成功: " + tableName);
            return true;
        } catch (Exception e) {
            System.err.println("同步失败 " + tableName + ": " + e.getMessage());
            return false;
        }
    }
    
    /**
     * 同步所有表
     */
    public void syncAllTables() {
        if (useDbJob) {
            // 使用数据库存储过程批量同步
            try {
                syncAllCall.execute();
                System.out.println("批量同步完成");
            } catch (Exception e) {
                System.err.println("批量同步失败: " + e.getMessage());
            }
        } else {
            // 使用Java循环同步
            for (String table : tables) {
                syncTable(table);
                try {
                    Thread.sleep(1000); // 表间等待1秒
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }
    
    /**
     * 手动触发同步
     */
    public void manualSync(List<String> tableNames) {
        if (tableNames == null || tableNames.isEmpty()) {
            syncAllTables();
        } else {
            for (String table : tableNames) {
                syncTable(table);
            }
        }
    }
    
    /**
     * 定时同步任务（如果不使用数据库作业）
     */
    @Scheduled(fixedDelayString = "${sync.interval}000")
    public void scheduledSync() {
        if (!useDbJob) {
            System.out.println("开始定时同步: " + new Date());
            syncAllTables();
            System.out.println("定时同步完成: " + new Date());
        }
    }
    
    /**
     * 获取同步状态
     */
    public Map<String, Object> getSyncStatus() {
        Map<String, Object> status = new LinkedHashMap<>();
        
        String sql = "SELECT C.TABLE_NAME, C.ENABLED, C.SYNC_MODE, " +
                    "C.SYNC_COLUMN, C.LAST_SYNC_TIME, " +
                    "S.LAST_SUCCESS_TIME, S.ROW_COUNT, S.LAST_ERROR_MSG " +
                    "FROM SYNC_CONFIG C " +
                    "LEFT JOIN SYNC_LAST_STATUS S ON C.TABLE_NAME = S.TABLE_NAME " +
                    "ORDER BY C.TABLE_NAME";
        
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        
        for (Map<String, Object> row : rows) {
            String tableName = (String) row.get("TABLE_NAME");
            status.put(tableName, row);
        }
        
        return status;
    }
}
```

### **4. 命令行控制器（不需要Web）**
```java
package com.dm.sync.controller;

import com.dm.sync.service.SimpleSyncService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
import java.util.Arrays;
import java.util.Map;
import java.util.Scanner;

@Component
public class ConsoleController implements CommandLineRunner {
    
    @Autowired
    private SimpleSyncService syncService;
    
    @Override
    public void run(String... args) {
        if (args.length > 0) {
            // 命令行参数模式
            handleArgs(args);
        } else {
            // 交互模式
            startInteractiveMode();
        }
    }
    
    private void handleArgs(String[] args) {
        switch (args[0].toLowerCase()) {
            case "sync":
                if (args.length > 1) {
                    // 同步指定表
                    syncService.manualSync(Arrays.asList(args).subList(1, args.length));
                } else {
                    // 同步所有表
                    syncService.syncAllTables();
                }
                break;
                
            case "status":
                Map<String, Object> status = syncService.getSyncStatus();
                printStatus(status);
                break;
                
            case "help":
                printHelp();
                break;
                
            default:
                System.out.println("未知命令，使用 'help' 查看帮助");
        }
    }
    
    private void startInteractiveMode() {
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("=== 达梦数据库同步工具 ===");
        System.out.println("输入 'help' 查看命令");
        
        while (true) {
            System.out.print("\n> ");
            String input = scanner.nextLine().trim();
            
            if (input.equalsIgnoreCase("exit") || input.equalsIgnoreCase("quit")) {
                System.out.println("退出程序");
                break;
            }
            
            String[] parts = input.split("\\s+");
            handleArgs(parts);
        }
        
        scanner.close();
    }
    
    private void printStatus(Map<String, Object> status) {
        System.out.println("\n=== 同步状态 ===");
        System.out.printf("%-20s %-8s %-10s %-15s %-20s %-10s%n", 
            "表名", "启用", "模式", "同步字段", "最后同步", "行数");
        System.out.println("-".repeat(100));
        
        for (Map.Entry<String, Object> entry : status.entrySet()) {
            Map<String, Object> row = (Map<String, Object>) entry.getValue();
            
            System.out.printf("%-20s %-8s %-10s %-15s %-20s %-10s%n",
                entry.getKey(),
                row.get("ENABLED"),
                row.get("SYNC_MODE"),
                row.get("SYNC_COLUMN") != null ? row.get("SYNC_COLUMN") : "-",
                row.get("LAST_SYNC_TIME") != null ? row.get("LAST_SYNC_TIME") : "-",
                row.get("ROW_COUNT") != null ? row.get("ROW_COUNT") : "0"
            );
        }
    }
    
    private void printHelp() {
        System.out.println("\n=== 可用命令 ===");
        System.out.println("sync [表名1 表名2 ...]  - 同步指定表（不指定则同步所有）");
        System.out.println("status                  - 查看同步状态");
        System.out.println("help                    - 显示此帮助");
        System.out.println("exit/quit               - 退出程序");
    }
}
```

### **5. 主应用类**
```java
package com.dm.sync;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class DmSyncSimpleApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(DmSyncSimpleApplication.class, args);
    }
}
```

## **四、部署和使用**

### **1. 项目结构（极简版）**
```
dm-sync-simple/
├── build.gradle.kts          # Gradle配置
├── src/main/
│   ├── java/com/dm/sync/
│   │   ├── config/
│   │   │   └── DataSourceConfig.java
│   │   ├── service/
│   │   │   └── SimpleSyncService.java
│   │   ├── controller/
│   │   │   └── ConsoleController.java
│   │   └── DmSyncSimpleApplication.java
│   └── resources/
│       └── application.yml
├── libs/
│   └── DmJdbcDriver18.jar
└── scripts/
    └── init_tables.sql
```

### **2. 一键初始化脚本**
```bash
#!/bin/bash
# init.sh - 一键初始化

echo "1. 创建数据库表..."
sqlplus SYSDBA/SYSDBA@localhost:5236 << EOF
-- 这里直接粘贴上面的SQL脚本
CREATE TABLE SYNC_CONFIG (...);
CREATE TABLE SYNC_LAST_STATUS (...);
CREATE OR REPLACE PROCEDURE PROC_SYNC_TABLE(...);
CREATE OR REPLACE PROCEDURE PROC_SYNC_ALL_TABLES(...);
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(...);
END;
/
-- 初始化表配置
BEGIN
    INSERT INTO SYNC_CONFIG (TABLE_NAME) VALUES ('USER_INFO');
    -- ... 其他表
    COMMIT;
END;
/
EOF

echo "2. 构建Java程序..."
./gradlew clean build

echo "3. 运行程序..."
java -jar build/libs/dm-sync-simple.jar
```

### **3. 使用方式**

**方式1：命令行模式**
```bash
# 同步所有表
java -jar dm-sync-simple.jar sync

# 同步指定表
java -jar dm-sync-simple.jar sync USER_INFO ORDER_INFO

# 查看状态
java -jar dm-sync-simple.jar status

# 交互模式
java -jar dm-sync-simple.jar
```

**方式2：后台服务模式**
```bash
# 后台运行，依赖数据库定时作业
nohup java -jar dm-sync-simple.jar > sync.log 2>&1 &
```

**方式3：手动触发（不启动定时任务）**
```java
// 可以写个简单的main方法手动触发
public class ManualSync {
    public static void main(String[] args) {
        SimpleSyncService service = new SimpleSyncService();
        service.syncAllTables();
    }
}
```

## **五、核心优势**

1. **极简架构**：只有1个配置表、1个状态表、2个存储过程
2. **智能处理**：
   - 自动检测时间字段 → 时间增量同步
   - 自动检测自增字段 → 键增量同步  
   - 什么都没有 → 全量同步
3. **零配置**：大部分表只需要配个表名，其他自动检测
4. **灵活部署**：命令行/后台服务/手动触发 多种方式
5. **资源占用少**：没有Web服务，没有复杂监控，内存占用小

## **六、扩展建议**

如果后续需要扩展，可以逐步添加：

1. **邮件通知**：同步失败时发送邮件
2. **简单Web界面**：Spring Boot加个Controller
3. **配置文件同步**：部分表不想在数据库配置，可以用YAML
4. **数据校验**：简单的count比对

但这个精简版已经足够覆盖十几张表的同步需求了！
