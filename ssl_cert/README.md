```bash
# 生成私钥
openssl genrsa -out ljy.com.key 2048

# 生成证书请求
openssl req -new -key ljy.com.key -out ljy.com.csr -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=LJY Company/OU=IT Department/CN=ljy.com"


# 生成证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ljy.com.key -out ljy.com.crt -subj "/C=CN/ST=Guangdong/L=Guangzhou/O=LJY Company/OU=IT Department/CN=ljy.com"
```
