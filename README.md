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
cat /etc/yum.repos.d/local.repo
[base]
name=base
baseurl=http://10.1.1.x/base
enabled=1
gpgcheck=0

yum -y install ansible docker
ansible -i hosts-list all -s -m copy -a "src=/etc/yum.repos.d/local.repo dest=/etc/yum.repos.d/local.repo"
ansible -i hosts-list all -s -m shell -a "systemctl disable firewalld"
ansible -i hosts-list all -s -m shell -a "systemctl stop firewalld"
ansible -i hosts-list all -s -m shell -a "setenforce 0"
ansible -i hosts-list all -s -m shell -a "sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config"
ansible -i hosts-list all -s -m shell -a "yum install -y wget git net-tools bind-utils iptables-services bridge-utilsbash-completion kexec-tools sos psacct vim lrzsz python-setuptools"
ansible -i hosts-list all -s -m shell -a "reboot"
ansible -i hosts-list all -s -m shell -a "yum install -y docker-1.12.6"
ansible -i hosts-list all -s -m shell -a "systemctl start docker"
ansible -i hosts-list all -s -m shell -a "systemctl enable docker"
```
4. Install Configuration Docker Repository
```
docker run -d -p 5000:5000 registry:2
tar xf docker-images.tar
cd docker-images/
for i in `ll|awk '{print $9}'`; do docker load < $i; done
docker images |grep"dataos.io"|awk '{print "docker tag "$3""$1":"$2}'| \
  sed-e s/registry.new.dataos.io/10.1.1.x:5000/| \
 xargs -i bash -c "{}"
docker images |grep "10.1.1.x:5000"| \
  awk'{print "docker push "$1":"$2}'| \
 xargs -i bash -c "{}"
```
5. Configuration Insecure-Registry
```
cat /etc/docker/daemon.json
{
    "insecure-registries": [
        "172.25.0.0/16",
        "10.1.1.x:5000",
        "docker-registry.default.svc:5000"
    ]
}
ansible -i hosts-list all -s -m copy -a "/etc/docker/daemon.json dest=/etc/docker/daemon.json"
ansible -i hosts-list all -s -m shell -a "systemctl restart docker"
```
