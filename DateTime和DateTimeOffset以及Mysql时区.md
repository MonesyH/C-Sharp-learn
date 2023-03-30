# 一、各自特征
1. DateTime和DateTimeOffset的最大区别在于是否包含时区信息。
* DateTimeOffset 含有相对utc的时区偏移量，如6/19/2008 7:00:00 AM +05:00；

* DateTime 含有时区，如 6/19/2008 2:00:00 AM Utc

所以如果需要在应用程序中处理多个不同的时区，使用DateTimeOffset可以更加方便和准确。

2. 此外，DateTime和DateTimeOffset在表示精度上也有所不同。DateTime的精度只能到Ticks，即百纳秒级别，而DateTimeOffset的精度可以达到Tick以下的纳秒级别。

# 二、 常用的DateTimeOffset 的构造
第一种：
```
DateTimeOffset.Now (根据本地时区生成偏移量，如东八区为2022/3/21 14:58:29 +08:00)
```

第二种：
```
new DateTimeOffset(2008, 6, 18, 7, 0, 0, new TimeSpan(-5, 0, 0));
```

第三种：
```
DateTime baseTime= new DateTime(2008, 6, 19, 7, 0, 0);

new DateTimeOffset(baseTime, TimeZoneInfo.Local.GetUtcOffset(baseTime));
```

# 三、 DateTimeOffset转换成DateTime
(1) ``` DateTime dateTime = DateTimeOffset.DateTime; ```

这样转换dateTime为dateTimeOffset的数值，时区丢失，即dateTime.Kind为UnSpecified

DateTime dateTime=DateTime.SpecifyKind(dateTimeOffset,DateTimeKind.Local);

这样转换DateTime为DateTimeOffset的数值，时区为Local(当然也可指定其它时区）

(2) ``` DateTime dateTime=DateTimeOffset.LocalTime; ```

根据本地时区换算为本地时间，且DateTime.Kind为Local

(3) ``` DateTime dateTime=dateTimeOffset.UtcDateTime; ```

根据本地时区换算为utc时间(0时区)，且dateTime.Kind为Utc

# 四、MySql的时区问题
起因是我发现插入到数据库的数据一直想差8小时

1. 是代码的问题（上面所说的DateTime和DateTimeOffset）
2. 是Mysql的问题

(1) 先去mysql看一下当前时间和本地时间是否匹配
``` select now(); ```

(2) 如果不匹配，则去查看时区是否为本地系统的
* docker：
   
   a . 进入容器 ``` docker exec -it mysql名称 bash``` 
   
   b. 查看当前时区 ``` date -R ```
   
   c. 修改时区 
   
``` 
cp /usr/share/zoneinfo/PRC /etc/localtime
# 或者
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# 退出
exit
# 重启容器生效
docker restart mysql5.7
 ```

* 本地：
```show variables like '%time_zone%';```
![时区](https://upload-images.jianshu.io/upload_images/20387877-70f51d67ff834903.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以我们现在要把 MySQL 的时区先给改对，可以通过修改配置文件来实现（/etc/mysql/mysql.conf.d/mysqld.cnf），如下：
![修改时区](https://upload-images.jianshu.io/upload_images/20387877-c1c2782ccc9af93c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

修改完成后，重启 MySQL，再来查看 MySQL 的时区：
![修改后](https://upload-images.jianshu.io/upload_images/20387877-9b21673bfd642b0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

