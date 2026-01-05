
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