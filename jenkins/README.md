# 备忘

获取初始化密码命令，也可以直接看映射的jenkins_home目录下的secrets/initialAdminPassword文件
```bash
docker-compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

安装初始化插件的时候由于网络问题，可能老是失败，重试一次看看，不行后面进入系统再重装