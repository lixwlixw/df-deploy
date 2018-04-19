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
   
createrepo /var/www/html/base/
createrepo /var/www/html/xxxx/
service httpd start
```     
2. Ensuring Host Access
```
   ssh-keygen -t rsa  
   for i in master1 \
    master2 \
    master3 \
    xxxxxx ; \
    do ssh-copy-id -i $i; \
    done
```     
3. Use Ansible Configuration Host
```
   
```
