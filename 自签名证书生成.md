# 自签名ssl证书生成

## 生成CA私钥

```shell
# 创建文件夹 ca 保存Ca相关
mkdir ca
cd ca
#创建私钥 (建议设置密码)
openssl genrsa -des3 -out myCA.key 2048
```

生成如下

```
Generating RSA private key, 2048 bit long modulus
.............................................+++
................................+++
e is 65537 (0x010001)
Enter pass phrase for myCA.key:
Verifying - Enter pass phrase for myCA.key:
```

## 生成CA证书

```shell
# 20 年有效期
openssl req -x509 -new -nodes -key myCA.key -sha256 -days 7300 -out myCA.crt
```

流程如下

```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Guang Dong
Locality Name (eg, city) []:ShenZhen
Organization Name (eg, company) [Internet Widgits Pty Ltd]:liuguang Inc
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:liuguang root CA
Email Address []:admin@liuguang.vip
```

> 把此证书导入需要部署的PC中即可，以后用此CA签署的证书都可以使用
>
> 查看证书信息命令 openssl x509 -in myCA.crt -noout -text

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            ce:32:70:80:74:a7:84:f1
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = CN, ST = Guang Dong, L = ShenZhen, O = liuguang Inc, CN = liuguang root CA, emailAddress = admin@liuguang.vip
        Validity
            Not Before: Sep 28 08:42:08 2018 GMT
            Not After : Sep 27 08:42:08 2023 GMT
        Subject: C = CN, ST = Guang Dong, L = ShenZhen, O = liuguang Inc, CN = liuguang root CA, emailAddress = admin@liuguang.vip
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a2:23:44:14:2b:77:89:61:16:88:17:f6:b3:fd:
                    88:e4:55:c3:2b:8d:1b:d7:25:81:34:e8:89:4d:70:
                    8c:a0:b2:80:98:7d:98:e5:65:5b:de:cb:cd:a5:0b:
                    86:8c:ff:00:27:24:22:b2:8a:69:4a:2d:ec:ff:f2:
                    82:cc:e9:7f:39:ce:9d:57:56:52:67:86:91:b0:39:
                    88:8c:e7:e3:73:f4:74:13:d9:64:3b:8c:19:49:74:
                    2f:25:57:20:af:7f:28:06:6f:8c:8b:69:b0:ed:b6:
                    2e:12:df:24:8e:54:89:56:8c:2a:4b:4f:35:ee:ca:
                    b6:f1:0f:8f:ca:50:21:f9:6f:81:00:01:29:3f:1c:
                    b2:7a:eb:f7:2e:f6:3d:03:00:e7:ae:5b:f9:08:8f:
                    90:7f:cd:5a:02:35:b9:ce:36:cb:ef:05:32:63:2b:
                    21:ba:3b:72:c5:56:b1:25:a9:4d:41:71:11:7e:b5:
                    0a:5f:7a:6f:0c:93:26:a7:71:93:d7:aa:c2:7d:1a:
                    5c:bd:0d:c2:7a:5f:12:86:73:0f:7b:48:8f:32:c8:
                    59:b8:0c:c8:69:b8:1f:1f:92:83:04:6c:04:75:96:
                    b7:36:6c:73:09:fe:91:ce:70:72:69:46:34:67:40:
                    09:fb:67:d1:e6:a1:ef:62:49:5b:a2:a8:e0:ef:aa:
                    34:c1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                AD:1C:AB:A0:85:7D:25:E4:09:55:8A:9E:30:68:14:5D:13:51:AD:61
            X509v3 Authority Key Identifier:
                keyid:AD:1C:AB:A0:85:7D:25:E4:09:55:8A:9E:30:68:14:5D:13:51:AD:61

            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         6b:df:b3:e8:bd:e1:b7:ae:43:e1:f4:e4:83:78:cc:09:04:32:
         2b:d8:9c:c5:ad:ac:e9:dc:8a:52:e6:cd:12:18:f8:9b:f5:00:
         5e:84:6c:7a:c5:19:4b:75:fc:81:a1:ec:e4:84:65:4c:cd:26:
         c2:a9:7c:f3:e3:b3:fb:19:97:47:02:af:3a:3a:ec:58:6a:87:
         ca:77:a4:a7:83:2d:b9:58:53:49:50:d1:b8:7f:3a:88:15:9b:
         24:d7:62:f3:05:4c:5e:80:cc:a2:52:5c:7b:c0:5c:0c:e1:88:
         e8:1b:6a:fb:e8:09:1c:7b:75:75:5c:f0:da:53:67:f5:f9:a9:
         ec:d8:9e:2c:13:5b:a7:9d:c3:ec:a9:58:92:cc:40:93:e0:ea:
         72:4c:3d:84:4f:bc:60:54:7e:13:26:2c:42:35:bf:44:90:04:
         57:ac:23:99:a8:1c:2a:ef:1d:81:14:c3:de:d4:df:23:11:2a:
         74:a9:11:55:bb:3f:c2:0a:12:be:c7:86:ec:ed:17:8b:3f:6c:
         0a:45:f8:5d:df:84:b9:08:b6:2a:20:6d:3a:6a:a4:21:8f:39:
         7c:92:b7:b7:e0:d1:12:53:84:f7:f6:ae:e7:6b:9d:65:7b:52:
         f4:4c:00:91:db:78:91:87:b1:d6:1f:cb:ab:a3:56:4b:96:f1:
         cc:83:ee:54
```

## 创建ssl证书私钥

```shell
cd ..
# 此文件夹存放待签名的证书
mkdir certs
cd certs
openssl genrsa -out localhost.key 2048
```

输出信息

```
Generating RSA private key, 2048 bit long modulus
...............+++
..............................................................................+++
e is 65537 (0x010001)
```

## 创建ssl证书CSR

```shell
openssl req -new -key localhost.key -out localhost.csr
```

输入相关信息

```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Guang Dong
Locality Name (eg, city) []:ShenZhen
Organization Name (eg, company) [Internet Widgits Pty Ltd]:liuguang Inc
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:liuguang cert
Email Address []:admin@liuguang.vip

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

## 创建域名附加配置文件

新建文件`cert.ext` 输入如下内容保存

```ini
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
IP.2 = 127.0.0.1
DNS.3 = test.com
DNS.4 = *.test.com
```

当前目录的文件

```
root@ubuntu:/home/liuguang/certs# ls -l
total 12
-rw-r--r-- 1 root root  237 Sep 28 08:57 cert.ext
-rw-r--r-- 1 root root 1050 Sep 28 08:51 localhost.csr
-rw------- 1 root root 1679 Sep 28 08:47 localhost.key
```

## 使用CA签署ssl证书

```shell
# ssl证书有效期10年
openssl x509 -req -in localhost.csr -out localhost.crt -days 3650 \
  -CAcreateserial -CA ../ca/myCA.crt -CAkey ../ca/myCA.key \
  -CAserial serial -extfile cert.ext
```

此步骤需要输入CA私钥的密码

```
Signature ok
subject=C = CN, ST = Guang Dong, L = ShenZhen, O = liuguang Inc, CN = liuguang cert, emailAddress = admin@liuguang.vip
Getting CA Private Key
Enter pass phrase for ../ca/myCA.key:
```

## 其它

查看签署的证书信息

```
root@ubuntu:/home/liuguang/certs# openssl x509 -in localhost.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            ad:38:b3:e5:12:d9:d5:dc
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = CN, ST = Guang Dong, L = ShenZhen, O = liuguang Inc, CN = liuguang root CA, emailAddress = admin@liuguang.vip
        Validity
            Not Before: Sep 28 09:50:04 2018 GMT
            Not After : Sep 25 09:50:04 2028 GMT
        Subject: C = CN, ST = Guang Dong, L = ShenZhen, O = liuguang Inc, CN = liuguang cert, emailAddress = admin@liuguang.vip
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:ae:03:e2:10:4a:0c:40:4e:94:c9:9b:67:03:6e:
                    4a:58:4c:23:c5:5c:c8:d8:ee:6e:7c:ee:76:16:cf:
                    bd:26:e8:bb:9d:54:6a:c2:de:30:69:a7:79:10:5f:
                    7d:03:47:5a:4a:0a:7d:48:39:15:58:5d:b2:6a:b1:
                    b8:a3:ba:56:1e:73:09:f4:28:6a:b6:19:ed:d3:41:
                    70:8a:4a:c0:a2:46:05:d2:64:50:a9:60:a5:5d:b1:
                    97:77:e8:ef:24:8f:1d:ce:7c:9c:e6:7c:5b:20:54:
                    ca:ee:0f:42:ff:0a:c8:ea:3d:8a:ac:44:92:ca:23:
                    e0:26:76:4d:43:1d:d0:de:ec:0d:5e:c6:65:0d:56:
                    65:1b:c2:70:82:fd:73:d1:57:d3:65:a2:5b:92:8c:
                    7f:74:ef:8f:78:b4:c6:c0:22:f3:09:cd:cd:ad:7b:
                    a5:cb:fc:c5:c3:1b:46:92:2f:6d:08:57:f9:02:ac:
                    90:3e:b1:48:ee:a7:cb:dd:3a:3f:4c:38:9c:fd:25:
                    94:ba:7f:58:8e:2c:e3:78:ef:9f:07:3a:5e:84:98:
                    5d:90:54:a1:06:83:94:36:d5:65:bf:09:12:d8:b0:
                    b3:f9:9a:7e:58:5b:ee:23:25:9e:2d:cc:0a:e0:a0:
                    be:35:5c:91:f9:a0:84:d7:09:ec:7d:f6:82:10:fb:
                    b4:9b
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                keyid:BA:E4:FF:29:AA:F6:66:C7:FA:E8:B9:86:5E:45:41:40:2F:D2:69:BE

            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature, Non Repudiation, Key Encipherment, Data Encipherment
            X509v3 Subject Alternative Name:
                DNS:localhost, IP Address:127.0.0.1, DNS:test.com, DNS:*.test.com
    Signature Algorithm: sha256WithRSAEncryption
         4f:f1:75:63:13:4b:d3:b6:7b:95:16:bf:84:6f:49:43:d2:55:
         0b:ef:81:32:06:4d:53:f4:1b:45:c1:b4:41:53:ee:6e:5b:03:
         a9:86:0f:7f:2b:c5:48:5e:9d:45:bf:9c:2b:45:b4:f3:d9:cc:
         f5:54:fd:27:c2:5d:9a:f5:1a:d9:26:ee:f0:57:84:a0:dc:5c:
         c2:5f:58:98:9c:57:af:a6:0c:38:7e:72:09:2c:e6:28:6e:d4:
         0b:ec:a5:f8:1c:ef:ce:17:dc:6b:23:40:a1:88:87:c3:8f:c9:
         63:3d:19:5c:c3:f5:b0:96:ba:93:3c:00:19:cb:a5:f0:45:37:
         38:04:49:45:81:c7:aa:ca:5c:92:b0:20:c9:0e:60:d5:8b:77:
         ed:e9:27:d4:93:02:15:9a:fa:82:21:71:bd:03:b0:34:bc:30:
         52:ad:08:46:7c:3b:70:4a:16:ac:f9:5b:45:a8:86:5f:23:3e:
         40:3d:f4:bb:78:47:cd:ca:79:67:09:d6:18:53:c8:7c:2b:01:
         86:5b:e8:74:06:78:09:05:6b:9a:bb:b3:d8:b0:77:92:bf:28:
         5c:43:e1:09:37:74:83:15:cb:b8:51:e1:12:ef:fa:ed:5f:20:
         99:b1:ae:d5:e7:d2:bd:fd:80:3e:ef:28:4d:21:a4:06:e9:29:
         1f:4d:1c:6f
```

使用CA验证一下证书是否通过

```
root@ubuntu:/home/liuguang/certs# openssl verify -CAfile ../ca/myCA.crt localhost.crt
localhost.crt: OK
```
