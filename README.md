# DF-Deploy
## 一.Planning
master x 3   
etcd x 3    
lb x 1    
node x 3  
origin-1.2 x 1    
service-brokers-etcd x 1    
glusterfs x 2    
heketi x 1    
nfs x 1    
yum mirror x 1    
docker mirror x 1    
dns x 1     


## 二. Preparation
    
1. Install Yum Repository  
```
   tar -xf yumrepo.tar -C /var/www/html/
   service httpd start
   createrepo /var/www/html/xxxx
```     
2. Ensuring Host Access
```
   ssh-keygen -t rsa
   
   for host in master1 \
    master2 \
    master3 \
    xxxxxx ; \
    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
    done
```     
  3 配置机器互信
     控制机生成key文件 ssh-keygen -t rsa
     配置控制机和其他机器互信  ssh-copy-id root@IP

  4 通过ansible 给所有机器配置yum客户端文件
