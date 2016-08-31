# 使用swiftclient 访问对象存储

## 创建swfit用户

swfit用户与RGW子用户对应，需要先创建普通用户

```
radosgw-admin user create --uid=johndoe --display-name="John Doe" --email=john@example.com
```

创建子用户

```
radosgw-admin subuser create --uid=johndoe --subuser=johndoe:swift --access=full
```

为子用户创建密码

```
radosgw-admin key create --subuser=johndoe:swift --key-type=swift --gen-secret
```

## 使用swift命令

创建bucket

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb post test
```

上传object

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb upload test myfile
```

下载object

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb download test myfile
```

列出集群中的容器\/对象

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb list 
```

删除容器\/对象

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb delete test [myfile]
```

查询容器\/对象信息

```
swift -V 1.0 -A http://server-201:7480/auth -U johndoe:swift -K ppJm1cQo\/XJ9WHQs6vSGuPefohwoQeKkp04ghYZb stat [test] [myfile]
```

