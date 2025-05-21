---
title: 腾讯云COS迁移到CloudFlare R2
date: "2025-05-21"
draft: false
tags: ["云计算"]
keywords: ["R2", "COS", "Rclone"]
categories: ["技术"]
---

早期考虑国内访问速度，选择了COS作为图片对象存储。24年开始，腾讯云[不再支持使用默认域名在浏览器预览](https://cloud.tencent.com/document/product/436/96243)，因为没有已备案的域名，度过了很久无法预览图片的日子。当数据基本准备就绪，在预估图片流量费用时，被腾讯云这复杂的计费方式和结算方式整懵了。  
对比过后，R2有明显的成本优势。  
- 免流出带宽费用（最大的优势），无论从R2下载多少数据，都不会产生额外流量费用
- 10G 免费存储空间，超出部分0.015 美元/GB/月（约 0.11 元/GB/月）  
- A 类操作（PUT/POST/DELETE）：4.5 美元/百万次；B 类操作（GET/HEAD）：0.36 美元/百万次；免费额度：100 万次 A 类操作 + 1000 万次 B 类操作/月
- 无 CDN 回源费用

### R2 Vs. COS  
| 费用项 | Cloudflare R2 | 腾讯云 COS |
| ----- | --------------| ---------- |
| 流出流量 | 免费 | 0.5元/GB |
| 存储(标准) | ~0.11元/GB/月 | 0.118 元/GB/月 |
| 请求(读写) | 0.36/4.5 美元/百万次 | 0.01 元/万次 |
| CDN 回源 | 免费 | 0.15 元/GB |


虽然产品目标用户是大陆用户，使用腾讯云访问速度会是最佳。但个人产品，没有很多用户，对图片访问速度的要求并不需要那么极致。能省一点是一点，计划从 COS 迁移到 R2。

## R2 迁移工具  
R2 提供了数据迁移，我没能成功运行，Cloudflare 也没有提供日志排查，无从下手。只能考虑备用方案 rclone。   

![迁移失败](https://img.mutoulbj.com/blog/r2-migrate-job.png)

## 使用 Rclone 进行迁移  
[Rclone](https://rclone.org/) 是一个非常强大云存储间进行数据同步、迁移的命令行工具。 

### 准备工作  
1. 分别在腾讯云和 Cloudflare 控制台申请各自的 AccessKey 与 AccessSecret; 
2. 按官网说明进行[安装](https://rclone.org/install/)。  

### 配置  
按以下示例，使用 `rclone config` 命令分别配置 cos 与 r2 远程存储。  
```bash
No remotes found, make a new one?
n) New remote
s) Set configuration password
q) Quit config
n/s/q> n
name> remote
Type of storage to configure.
Choose a number from below, or type in your own value
[snip]
XX / Amazon S3 Compliant Storage Providers including AWS, ...
   \ "s3"
[snip]
Storage> s3
Choose your S3 provider.
Choose a number from below, or type in your own value
 1 / Amazon Web Services (AWS) S3
   \ "AWS"
 2 / Ceph Object Storage
   \ "Ceph"
 3 / DigitalOcean Spaces
   \ "DigitalOcean"
 4 / Dreamhost DreamObjects
   \ "Dreamhost"
 5 / IBM COS S3
   \ "IBMCOS"
 6 / Minio Object Storage
   \ "Minio"
 7 / Wasabi Object Storage
   \ "Wasabi"
 8 / Any other S3 compatible provider
   \ "Other"
provider> 1
Get AWS credentials from runtime (environment variables or EC2/ECS meta data if no env vars). Only applies if access_key_id and secret_access_key is blank.
Choose a number from below, or type in your own value
 1 / Enter AWS credentials in the next step
   \ "false"
 2 / Get AWS credentials from the environment (env vars or IAM)
   \ "true"
env_auth> 1
AWS Access Key ID - leave blank for anonymous access or runtime credentials.
access_key_id> XXX
AWS Secret Access Key (password) - leave blank for anonymous access or runtime credentials.
secret_access_key> YYY
Region to connect to.
Choose a number from below, or type in your own value
   / The default endpoint - a good choice if you are unsure.
 1 | US Region, Northern Virginia, or Pacific Northwest.
   | Leave location constraint empty.
   \ "us-east-1"
   / US East (Ohio) Region
 2 | Needs location constraint us-east-2.
   \ "us-east-2"
   / US West (Oregon) Region
 3 | Needs location constraint us-west-2.
   \ "us-west-2"
   / US West (Northern California) Region
 4 | Needs location constraint us-west-1.
   \ "us-west-1"
   / Canada (Central) Region
 5 | Needs location constraint ca-central-1.
   \ "ca-central-1"
   / EU (Ireland) Region
 6 | Needs location constraint EU or eu-west-1.
   \ "eu-west-1"
   / EU (London) Region
 7 | Needs location constraint eu-west-2.
   \ "eu-west-2"
   / EU (Frankfurt) Region
 8 | Needs location constraint eu-central-1.
   \ "eu-central-1"
   / Asia Pacific (Singapore) Region
 9 | Needs location constraint ap-southeast-1.
   \ "ap-southeast-1"
   / Asia Pacific (Sydney) Region
10 | Needs location constraint ap-southeast-2.
   \ "ap-southeast-2"
   / Asia Pacific (Tokyo) Region
11 | Needs location constraint ap-northeast-1.
   \ "ap-northeast-1"
   / Asia Pacific (Seoul)
12 | Needs location constraint ap-northeast-2.
   \ "ap-northeast-2"
   / Asia Pacific (Mumbai)
13 | Needs location constraint ap-south-1.
   \ "ap-south-1"
   / Asia Pacific (Hong Kong) Region
14 | Needs location constraint ap-east-1.
   \ "ap-east-1"
   / South America (Sao Paulo) Region
15 | Needs location constraint sa-east-1.
   \ "sa-east-1"
region> 1
Endpoint for S3 API.
Leave blank if using AWS to use the default endpoint for the region.
endpoint>
Location constraint - must be set to match the Region. Used when creating buckets only.
Choose a number from below, or type in your own value
 1 / Empty for US Region, Northern Virginia, or Pacific Northwest.
   \ ""
 2 / US East (Ohio) Region.
   \ "us-east-2"
 3 / US West (Oregon) Region.
   \ "us-west-2"
 4 / US West (Northern California) Region.
   \ "us-west-1"
 5 / Canada (Central) Region.
   \ "ca-central-1"
 6 / EU (Ireland) Region.
   \ "eu-west-1"
 7 / EU (London) Region.
   \ "eu-west-2"
 8 / EU Region.
   \ "EU"
 9 / Asia Pacific (Singapore) Region.
   \ "ap-southeast-1"
10 / Asia Pacific (Sydney) Region.
   \ "ap-southeast-2"
11 / Asia Pacific (Tokyo) Region.
   \ "ap-northeast-1"
12 / Asia Pacific (Seoul)
   \ "ap-northeast-2"
13 / Asia Pacific (Mumbai)
   \ "ap-south-1"
14 / Asia Pacific (Hong Kong)
   \ "ap-east-1"
15 / South America (Sao Paulo) Region.
   \ "sa-east-1"
location_constraint> 1
Canned ACL used when creating buckets and/or storing objects in S3.
For more info visit https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html#canned-acl
Choose a number from below, or type in your own value
 1 / Owner gets FULL_CONTROL. No one else has access rights (default).
   \ "private"
 2 / Owner gets FULL_CONTROL. The AllUsers group gets READ access.
   \ "public-read"
   / Owner gets FULL_CONTROL. The AllUsers group gets READ and WRITE access.
 3 | Granting this on a bucket is generally not recommended.
   \ "public-read-write"
 4 / Owner gets FULL_CONTROL. The AuthenticatedUsers group gets READ access.
   \ "authenticated-read"
   / Object owner gets FULL_CONTROL. Bucket owner gets READ access.
 5 | If you specify this canned ACL when creating a bucket, Amazon S3 ignores it.
   \ "bucket-owner-read"
   / Both the object owner and the bucket owner get FULL_CONTROL over the object.
 6 | If you specify this canned ACL when creating a bucket, Amazon S3 ignores it.
   \ "bucket-owner-full-control"
acl> 1
The server-side encryption algorithm used when storing this object in S3.
Choose a number from below, or type in your own value
 1 / None
   \ ""
 2 / AES256
   \ "AES256"
server_side_encryption> 1
The storage class to use when storing objects in S3.
Choose a number from below, or type in your own value
 1 / Default
   \ ""
 2 / Standard storage class
   \ "STANDARD"
 3 / Reduced redundancy storage class
   \ "REDUCED_REDUNDANCY"
 4 / Standard Infrequent Access storage class
   \ "STANDARD_IA"
 5 / One Zone Infrequent Access storage class
   \ "ONEZONE_IA"
 6 / Glacier Flexible Retrieval storage class
   \ "GLACIER"
 7 / Glacier Deep Archive storage class
   \ "DEEP_ARCHIVE"
 8 / Intelligent-Tiering storage class
   \ "INTELLIGENT_TIERING"
 9 / Glacier Instant Retrieval storage class
   \ "GLACIER_IR"
storage_class> 1
Remote config
Configuration complete.
Options:
- type: s3
- provider: AWS
- env_auth: false
- access_key_id: XXX
- secret_access_key: YYY
- region: us-east-1
- endpoint:
- location_constraint:
- acl: private
- server_side_encryption:
- storage_class:
Keep this "remote" remote?
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d>
```   

配置完成后，使用 `rclone config show` 检查配置是否正确。  
```bash
~$ rclone config show
[cos]
type = s3
provider = TencentCOS
access_key_id = access_key
secret_access_key = secret_key
endpoint = cos.ap-shanghai.myqcloud.com
acl = default
storage_class = STANDARD

[r2]
type = s3
provider = Cloudflare
access_key_id = access_key
secret_access_key = secret_key
endpoint = https://xxxxxxx.r2.cloudflarestorage.com
```

### 开始备份  
一行命令即可进行备份。  
```bash
nohup rclone copy cos:[桶名]/ r2:[桶名]/ --progress --transfers=2 --checkers=4 --stats=20s > rclone.log &
```

参数解释：
- `--process` 查看进度
- `--chekers` 预检查的文件数量,根据服务器配置进行合理设置
- `--transfers` 同时传输的文件数量  
- `--stats` 日志输出的时间间隔  

Rclone 提供了大量的参数用于个性化配置，具体的参照文档说明。 

### 解决OOM 问题  
使用很简单，但在同步200多万图片，存储达到100G时，运行一段时间后会 OOM 而挂掉。Rclone支持续传，再次运行可以继续同步。更好的方式是使用[Big syncs with millons of files](https://github.com/rclone/rclone/wiki/Big-syncs-with-millions-of-files) 提供的方案。  

- 读取所有的对象名称  
```bash
rclone lsf --files-only -R src:bucket | sort > src
rclone lsf --files-only -R dst:bucket | sort > dst
```
- 使用 comm 查找需要传输的文件/对象  
```bash
comm -23 src dst > need-to-transfer
comm -13 src dst > need-to-delete
```
- 进行分块传输  
```bash
rclone copy src:bucket dst:bucket --files-from need-to-transfer --no-traverse
rclone delete src:bucket dst:bucket --files-from need-to-delete --no-traverse
```

### 参考资料    
1. https://forum.rclone.org/t/rclone-oom-killed-during-sync-files-check-error-137/33798/17  
2. https://forum.rclone.org/t/s3-sync-memory-overflow-leads-to-oom/44153  
3. https://github.com/rclone/rclone/issues/7974  
4. https://github.com/rclone/rclone/wiki/Big-syncs-with-millions-of-files  
5. https://rclone.org/docs/  
