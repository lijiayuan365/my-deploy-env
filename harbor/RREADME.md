```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz
tar -xvf harbor-offline-installer-v2.10.0.tgz
cd harbor
```
## 配置模板
```bash
cp harbor.yml.tmpl harbor.yml
```


## 将证书放进去，修改harbor.yml文件证书绝对位置
```bash
certificate: /your-path/ssl_cert/ljy.com.crt
private_key: /your-path/ssl_cert/ljy.com.key
```

## 安装
```bash
sudo ./install.sh
```

