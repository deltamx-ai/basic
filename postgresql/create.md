使用podman创建 postgresql 容器，并启动

```
podman run -d \
  --name postgres17 \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=MyStrongPassword123 \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v pg17_data:/var/lib/postgresql/data \
  docker.io/library/postgres:17
```

    查看容器运行状态和启动日志：
    Bash

    podman ps
    podman logs postgres17

    日志最后出现 database system is ready to accept connections 即代表启动成功[8]。

    进入容器使用 psql 命令行连接测试：
    Bash

    podman exec -it postgres17 psql -U myuser -d mydb

    外部工具（如 Windows 上的 DBeaver / DataGrip / pgAdmin）连接参数：

        主机 (Host)：localhost（或 127.0.0.1）

        端口 (Port)：5432

        数据库 (Database)：mydb

        用户名 (User)：myuser

        密码 (Password)：MyStrongPassword123

    停止 / 启动容器：
    Bash

    podman stop postgres17
    podman start postgres17

注意事项

    密码和用户名仅在首次初始化时生效：POSTGRES_USER 和 POSTGRES_PASSWORD 只在数据卷 pg17_data 为空（首次建库）时才会执行设置[9]。如果以后需要修改密码，需要进入数据库执行 SQL：
    Bash

    podman exec -it postgres17 psql -U myuser -d mydb -c "ALTER USER myuser WITH PASSWORD 'NewPassword123';"

    想彻底重置重来：如果测试时想删除所有数据重新初始化，需要同时删掉容器和卷：
    Bash

    podman rm -f postgres17
    podman volume rm pg17_data
