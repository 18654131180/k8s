###在kubernetes中搭建LNMP环境，并安装ecshop


首先克隆本项目：https://github.com/18654131180/k8s.git

####下载镜像
```
docker pull mysql:5.7
docker pull registry.cn-shanghai.aliyuncs.com/zhouyu-docker/nginx-php-fpm
```

####用dockerfile重建nginx-php-fpm镜像
```
cd k8s_ecshop/dz_web_dockerfile/
docker build -t nginx-php .

```

####将镜像push到harbor
```
##登录harbor，并push新的镜像
docker login 192.168.36.132 //输入正确的用户名和密码 admin/Harbor12345
docker tag nginx-php 192.168.36.132/base/nginx-php
docker push 192.168.36.132/base/nginx-php
docker tag  mysql:5.7  192.168.36.132/base/mysql:5.7
docker push 192.168.36.132/base/mysql:5.7
```

####搭建NFS

假设kubernetes集群网段为172.7.5.0/24，本机IP为172.7.5.13
* 安装包
```
yum install nfs-utils
```
* 编辑配置文件
```
vim /etc/exportfs  //内容如下
/data/k8s/ 172.7.5.0/24(sync,rw,no_root_squash)
```
* 启动服务
```
systemctl start nfs
systemctl enable nfs
```
* 创建目录
```
mkdir -p  /data/k8s/ecshop_web/{appserver,ecshop}
```

####搭建MySQL服务
* 创建secret (设定mysql的root密码)
```
kubectl create secret generic mysql-pass --from-literal=password=123456
```
* 创建pv
```
cd ../../k8s_ecshop/mysql
kubectl create -f mysql-pv.yaml
```
* 创建pvc
```
kubectl create -f mysql-pvc.yaml
```
* 创建deployment
```
kubectl create -f mysql-dp.yaml 
```
* 创建service
```
kubectl create -f mysql-svc.yaml
```

####搭建Nginx+php-fpm服务
* 搭建pv
```
cd ../../k8s_ecshop/nginx_php
kubectl create -f web-pv.yaml
```
* 创建pvc
```
kubectl create -f web-pvc.yaml
```
* 创建deployment
```
kubectl create -f web-dp.yaml 
```
* 创建service
```
kubectl create -f web-svc.yaml
```
* 设置MySQL普通用户
```
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -p123456  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by '123456';
```
* 设置Nginx代理
```
注意：目前nginx服务是运行在kubernetes集群里，node节点以及master节点上是可以通过cluster-ip访问到，但是外部的客户端就不能访问了。
      所以，可以在任意一台node或者master上建一个nginx反向代理即可访问到集群内的nginx。
kubectl get svc dz-web //查看cluster-ip，我的ip是10.68.190.99
nginx代理配置文件内容如下：
server {
            listen 80;
            server_name *.*.*;

            location / {
                proxy_pass      http://10.68.190.99:80;
                proxy_set_header Host   $host;
                proxy_set_header X-Real-IP      $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

```
做完Nginx代理，就可以通过node的IP来访问了。
```
