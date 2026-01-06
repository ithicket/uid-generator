
# 部署postgresql
```shell
docker run -d \
  --name postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=pg@2025 \
  -p 5432:5432 \
  postgres:latest

docker exec -it postgres psql -U postgres

-- 1. 创建用户（设置密码，示例：username = myuser, password = mypassword）
CREATE USER myuser WITH PASSWORD 'mypassword';

-- 2. 创建数据库（示例：dbname = mydb，指定所有者为 myuser）
CREATE DATABASE mydb OWNER myuser;

-- 3. 连接到新创建的数据库（必须先连接，否则 Schema 创建在 postgres 数据库中）
\c mydb;

-- 4. 创建 Schema（示例：schema = myschema，指定所有者为 myuser）
CREATE SCHEMA myschema AUTHORIZATION myuser;

-- 5. 授权：给用户赋予数据库的全部权限（可选，根据需求调整）
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;

-- 6. 授权：给用户赋予 Schema 的全部权限
GRANT ALL PRIVILEGES ON SCHEMA myschema TO myuser;

-- 7. 授权：给用户赋予 Schema 下所有表的操作权限（未来创建的表也生效）
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT ALL ON TABLES TO myuser;

-- 退出 PostgreSQL 命令行
\q

```


# 创建表
```sql
DROP TABLE IF EXISTS worker_node;
CREATE TABLE worker_node (
     id BIGSERIAL PRIMARY KEY,
     host_name VARCHAR(64) NOT NULL,
     port VARCHAR(64) NOT NULL,
     type INTEGER NOT NULL,
     launch_date DATE NOT NULL,
     modified TIMESTAMPTZ NOT NULL,
     created TIMESTAMPTZ NOT NULL
);

COMMENT ON TABLE worker_node IS 'DB WorkerID Assigner for UID Generator';
COMMENT ON COLUMN worker_node.id IS 'auto increment id';
COMMENT ON COLUMN worker_node.host_name IS 'host name';
COMMENT ON COLUMN worker_node.port IS 'port';
COMMENT ON COLUMN worker_node.type IS 'node type: ACTUAL or CONTAINER';
COMMENT ON COLUMN worker_node.launch_date IS 'launch date';
COMMENT ON COLUMN worker_node.modified IS 'modified time';
COMMENT ON COLUMN worker_node.created IS 'created time';
```

# SpringBoot 集成

1、依赖
```xml
<dependency>
    <groupId>com.baidu.fsg</groupId>
    <artifactId>uid-generator</artifactId>
    <version>1.0.17</version>
</dependency>
```

2、添加扫描
```java
@MapperScan(basePackages = "com.baidu.fsg.uid.mapper")
@SpringBootApplication
public class Application {

}
```

```yml
mybatis-plus:
  # 扫描Mapper接口（指定JAR中的Mapper包路径，多个包用逗号分隔）
  type-aliases-package: com.baidu.fsg.uid.domain
  # 扫描XML映射文件（关键：用classpath*: 代替 classpath:，支持扫描所有类路径（包括外部JAR））
  mapper-locations:
    - classpath:mapper/*.xml                          # 当前项目的XML文件
    - classpath*:/META-INF/mybatis/mapper/*.xml       # JAR中的XML文件
```

3、配置类
```java
package com.demo.common.config;


import com.baidu.fsg.uid.generator.CachedUidGenerator;
import com.baidu.fsg.uid.mapper.WorkerNodeMapper;
import com.baidu.fsg.uid.service.impl.DisposableWorkerIdAssigner;
import jakarta.annotation.Resource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class UidGeneratorConfig {

    @Resource
    private WorkerNodeMapper workerNodeMapper;

    @Bean
    public DisposableWorkerIdAssigner disposableWorkerIdAssigner() {
        return new DisposableWorkerIdAssigner(workerNodeMapper);
    }

    @Bean
    public CachedUidGenerator cachedUidGenerator(DisposableWorkerIdAssigner disposableWorkerIdAssigner) {
        CachedUidGenerator generator = new CachedUidGenerator();
        generator.setWorkerIdAssigner(disposableWorkerIdAssigner);

        // 时间位配置
        generator.setTimeBits(30);
        generator.setWorkerBits(22);
        generator.setSeqBits(11);
        generator.setEpochStr("2026-01-01");

        // 缓存配置
        generator.setBoostPower(3);
        generator.setScheduleInterval(60L);

        return generator;
    }

}
```

4、使用
```java
package com.demo.controller;

import com.baidu.fsg.uid.generator.UidGenerator;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RequiredArgsConstructor
@RestController
@RequestMapping("/system/uid")
public class UidController {

    private final UidGenerator cachedUidGenerator;

    @GetMapping("/generateUid")
    public long generateUid() {
        return cachedUidGenerator.getUID();
    }

}
```

