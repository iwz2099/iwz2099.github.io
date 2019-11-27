# mac下openssl签发私有证书


####  生成客户端私钥:
```
openssl genrsa -out server.key 2048
```

#### 生成客户端证书：
```
openssl req -new -sha256 -x509 -days 3650 -key server.key -out server.crt
```

#### 生成证书申请文件：
```
openssl req -new -key server.key -out server.csr
```
<!-- more -->
####  生成CA私钥
采用des3加密，需要输入4位以上密码：
```
openssl genrsa -des3 -out ca.key 4096
```

#### 生成CA证书
```
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt
```

> CA配置配置文件/private/etc/ssl/openssl.cnf(我这里是mac系统)添加如下配置

```
[ ca ]
default_ca       = ca_default
[ ca_default ]
dir          = /etc/ssl/diyca # 指定了CA的根目录
certs        = $dir/certs            # 已经签发的证书的存储目录
crl_dir      = $dir/crl            # 存储证书吊销列表的目录
database     = $dir/index.txt      # 数据库的索引文件，用来存放签发证书的信息。
#unique_subject  = no               #设置为’no’表示允许同时创建多个相同主题的证书。
new_certs_dir = $dir/newcerts        # 设置存放新签发的证书的默认位置
Certificate   = $dir/cacert.pem # 指定CA证书
serial        = $dir/serial          # 指定存放当前序列号的文件,写入00即可
crl           = $dir/crl.pem         # 当前的CRL
private_key   = $dir/private/cakey.pem # CA的私钥
default_md = md5
RANDFILE      = $dir/private/.rand   #指明一个用来读写时候产生random key的seed文件。
policy= policy_match

[ policy_match ]
countryName= match
stateOrProvinceName= match
organizationName= match
organizationalUnitName= optional
commonName= supplied
emailAddress= optional

[ policy_anything ]
countryName = optional
stateOrProvinceName= optional
localityName= optional
organizationName = optional
organizationalUnitName = optional
commonName= supplied
emailAddress= optional
```


#### 本地CA签发证书：
```
openssl ca -in server.csr -out server.crt -cert ca.crt -keyfile ca.key -days 3650
```
server.pem就是签名后的证书
```
cp /etc/ssl/diyca/newcerts/00.pem server.pem
```

下面拿着server.key和server.pem,到web服务器上部署即可。

