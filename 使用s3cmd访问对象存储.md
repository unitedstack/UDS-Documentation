# 使用s3cmd访问对象存储

## 1 前言

UnitedStack 提供的对象存储服务，支持 S3 和 Swift 兼容的两套接口。下面将介绍如何使用基于命令行的 S3 服务客户端s3cmd 访问对象存储服务。

## 2 配置 s3 客户端

2.1 在linux上安装s3客户端

s3cmd 是由 Python编写，因此可以通过 pip 来进行安装:

```
# pip install s3cmd
```

此外可以通过发行版自带的包管理器安装,下面以 CentOS为例:

```
# yum install s3cmd
```

2.2 在Mac OSX上安装s3客户端

在 Mac OSX,同样可以通过 pip安装，或者使用包管理器 Homebrew 进行安装:

```
# brew install s3cmd
```

执行下面的命令来检查安装是否成功:

```
# s3cmd --version
s3cmd version 1.5.2
```

如果可以显式 s3cmd 的版本，则说明安装成功。

## 3 创建 s3 账号

由于对象存储的前端还在开发当中，所以用户的创建需要由 UnitedStack 的支持工程师来完成。每个用户的唯一标示是两个 key:

* access\_key
* secret\_key

3.1 配置 s3cmd

在当前用户的家目录下创建 .s3cfg 文件\(也是 s3cmd 默认配置文件路径\)，并填入以下内容:

```
[default]
access_key = your_access_key
bucket_location = US
cloudfront_host = s3-sh.ustack.com
cloudfront_resource = /2010-07-15/distribution
default_mime_type = binary/octet-stream
delete_removed = False
dry_run = False
encoding = UTF-8
encrypt = False
follow_symlinks = False
force = False
get_continue = False
gpg_command = /usr/bin/gpg
gpg_decrypt = %(gpg_command)s -d --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_encrypt = %(gpg_command)s -c --verbose --no-use-agent --batch --yes --passphrase-fd %(passphrase_fd)s -o %(output_file)s %(input_file)s
gpg_passphrase =
guess_mime_type = True
host_base = s3-sh.ustack.com[:port]
host_bucket = s3-sh.ustack.com
human_readable_sizes = False
list_md5 = False
log_target_prefix =
preserve_attrs = True
progress_meter = True
proxy_host =
proxy_port = 0
recursive = False
recv_chunk = 4096
reduced_redundancy = False
secret_key = your_secret_key
send_chunk = 96
simpledb_host = sdb.amazonaws.com
skip_existing = False
socket_timeout = 300
urlencoding_mode = normal
use_https = False
verbosity = WARNING
signature_v2 = True 
```

以上默认有5个字段需要确认，其中和认证相关的2个字段:

1. access\_key
2. secret\_key

需要和创建 s3 账号时的一致，如果提供的 key 中包含了反斜线，它们是JSON的转义字符，只删除反斜线就可以了。剩下3个字段和访问 S3 服务域名有关:

1. cloudfront\_host
2. host\_base
3. host\_bucket

## 4 使用对象存储

配置完 s3cmd 就可以通过它来使用对象存储了。

对象存储中有两个非常重要的概念，bucket 和 object。object 对应需要存储的文件，而 bucket 作为 object 的存储空间。所以对象存储的操作主要涉及到的就是对 bucket 和 object 的操作。

4.1 操作 bucket

创建 bucket 的命令格式:

```
# s3cmd mb s3://BUCKET
```

以创建名为 test\_bucket\_1 的 bucket 为例:

```
# s3cmd mb s3://test-bucket_1
Bucket 's3://test-bucket_1/' created
```

列举 bucket 的命令格式:

```
# s3cmd ls
```

以列举当前的bucket 为例:

```
# s3cmd ls
2015-08-12 03:56  s3://test-bucket_1
```

可以看到我们刚才创建的 test\_bucket\_1

获取Bucket位置命令格式：

```
# s3cmd info s3://BUCKET
```

获取当前bucket位置信息：

```
# s3cmd info s3://test-bucket-1
s3://test-bucket-1/ (bucket):
   Location:  us-east-1
ERROR: S3 error: None
   Expiration Rule: none
   policy: <?xml version="1.0" encoding="UTF-8"?><ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Name>test-bucket-1</Name><Prefix></Prefix><Marker></Marker><MaxKeys>1000</MaxKeys><IsTruncated>false</IsTruncated><Contents><Key>test1.txt</Key><LastModified>2015-08-14T02:48:39.000Z</LastModified><ETag>&quot;4b49d7dd076b0b71e0eda307388fac57&quot;</ETag><Size>348</Size><StorageClass>STANDARD</StorageClass><Owner><ID>mikulely</ID><DisplayName>Jiaying Ren</DisplayName></Owner></Contents></ListBucketResult>
```

删除 bucket的命令格式:

```
# s3cmd rb s3://BUCKET
```

以删除我们刚创建的 test\_bucket\_1 为例:

```
# s3cmd rb s3://test-bucket_1
Bucket 's3://test-bucket_1/' removed
```

4.2 操作 object

需要在 S3 中存储的文件在对象存储中被称为 object。为了说明上传 object 的过程，首先我们来创建一个用来上传的文件 test.txt \(也就是一个object\),并写入 samplecontent作为文件的内容:

```
# cat test.txt
samplecontent
```

由于 object 必须属于一个bucket，接下来需要创建保存 object 的 bucket:

```
# s3cmd mb s3://test-bucket_2
Bucket 's3://test-bucket_2/' created
```

上传 object 的命令格式:

```
s3cmd put FILE [FILE...] s3://BUCKET[/PREFIX]
```

以上传刚创建的 test.txt 文件到 test\_bucket\_2 bucket 为例:

```
# s3cmd put test.txt s3://test-bucket_2
WARNING: Module python-magic is not available. Guessing MIME types based on file extensions.
test.txt -> s3://test-bucket_2/test.txt  [1 of 1]
 14 of 14   100% in    0s    20.59 kB/s
 14 of 14   100% in   90s     0.16 B/s  done
```

列出 bucket 中 object 的命令格式:

```
# s3cmd ls [s3://BUCKET[/PREFIX]]
```

以列出当前 bucket 中的 object:

```
# s3cmd ls s3://test-bucket_2
2015-08-12 04:22        14   s3://test-bucket_2/test.txt
```

下载bucket中object的命令格式：

```
# s3cmd get s3://BUCKET/OBJECT LOCAL_FILE
```

下载当前bucket中的文件test.txt，并本地命名localtest.txt：

```
# s3cmd get s3://test-bucket-2/test.txt  localtest.txt
s3://test-bucket-2/test.txt -> localtest.txt  [1 of 1]
s3://test-bucket-2/test.txt -> localtest.txt  [1 of 1]
 348 of 348   100% in    0s    10.58 kB/s  done
```

删除bucket中object的命令格式：

```
# s3cmd del s3://BUCKET/FILENAME
```

删除当前bucket中的test.txt对象：

```
# s3cmd del s3://test-bucket-2/test.txt
File s3://test-bucket-2/test.txt delete
```

拷贝bucket中object的命令格式：

```
# s3cmd cp s3://BUCKET1/OBJECT1 s3://BUCKET2[/OBJECT2]
```

拷贝对象，从一个bucket到另一个bucket：

```
# s3cmd cp s3://test-bucket-2/test1.txt s3://test-bucket-1/test1.txt
WARNING: Retrying failed request: /test1.txt ()
WARNING: Waiting 3 sec...
File s3://test-bucket-2/test1.txt copied to s3://test-bucket-1/test1.txt
```

获取object信息命令格式：

```
# s3cmd info s3://BUCKET/OBJECT
```

获取当前object的信息：

```
# s3cmd info s3://test-bucket-2/test1.txt
s3://test-bucket-2/test1.txt (object):
   File size: 348
   Last mod:  Fri, 14 Aug 2015 02:02:37 GMT
   MIME type: text/plain
   MD5 sum:   4b49d7dd076b0b71e0eda307388fac57
   SSE:       NONE
   policy: <?xml version="1.0" encoding="UTF-8"?><ListBucketResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><Name>test-bucket-2</Name><Prefix></Prefix><Marker></Marker><MaxKeys>1000</MaxKeys><IsTruncated>false</IsTruncated><Contents><Key>test1.txt</Key><LastModified>2015-08-14T02:02:37.000Z</LastModified><ETag>&quot;4b49d7dd076b0b71e0eda307388fac57&quot;</ETag><Size>348</Size><StorageClass>STANDARD</StorageClass><Owner><ID>mikulely</ID><DisplayName>Jiaying Ren</DisplayName></Owner></Contents></ListBucketResult>
```

