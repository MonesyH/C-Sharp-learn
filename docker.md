### Docker配置

#### **docker 配置Azure Sql Edge**

```bash
sudo docker pull mcr.microsoft.com/azure-sql-edge:latest

sudo docker run --cap-add SYS_PTRACE -e 'ACCEPT_EULA=1' -e 'MSSQL_SA_PASSWORD=[password]' -p 1433:1433 --name azuresqledge -d mcr.microsoft.com/azure-sql-edge

password为项目中配置的密码
```

以下是运行成功的效果：

![image-20221108071815010](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108071815010.png)

![image-20221108071723132](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108071723132.png)

![image-20221108072049176](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108072049176.png)

---

#### **docker 配置RabbitMQ**

终端运行：

```bash
docker pull rabbitmq

docker run \

-e RABBITMQ_DEFAULT_USER=guest \

-e RABBITMQ_DEFAULT_PASS=guest \

--name mq \

--hostname localhost \

-p 15672:15672 \

-p 5672:5672 \

-d \

rabbitmq:3-management
```

以下是运行成功的效果：

![image-20221108071516571](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108071516571.png)

![image-20221108071447252](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108071447252.png)

---

#### docker配置redis

```bash
拉去官方镜像：
docker pull redis:latest
```

![img](https://www.runoob.com/wp-content/uploads/2016/06/docker-redis3.png)

```bash
查看本地镜像中是否已经存在：
docker images
```

![img](https://www.runoob.com/wp-content/uploads/2016/06/docker-redis4.png)

```bash
运行 redis 容器:
docker run -itd --name redis-test -p 6379:6379 redis
```

![img](https://www.runoob.com/wp-content/uploads/2016/06/docker-redis5.png)	

![image-20221108072709371](/Users/chensibin/Library/Application Support/typora-user-images/image-20221108072709371.png)

---

#### docker配置seq

```bash
拉去镜像：
docker pull datalust/seq
```

```bash
PH=$(echo '<password>' | docker run --rm -i datalust/seq config hash)

docker run \
  --name seq \
  -d \
  --restart unless-stopped \
  -e ACCEPT_EULA=Y \
  -e SEQ_FIRSTRUN_ADMINPASSWORDHASH="$PH" \
  -v /path/to/seq/data:/data \
  -p 80:80 \
  -p 5341:5341 \
  datalust/seq
```

---

#### docker配置MySQL

```bash
1、mac m1下载mysql镜像
docker pull --platform linux/x86_64 mysql:5.7

2、启动容器
docker run -itd --name mysql-test -v /Users/{username}/mysql-data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7

如报没有权限就在开头部分加上sudo指令

`/Users/{username}/...` 这部分中的{username}需要更换成自己电脑username
```

