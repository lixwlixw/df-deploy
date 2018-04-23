# DF-Deploy
## 一. Planning
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
6. Configuration Docker Storage
```
ansible -i hosts-list node -s -m shell -a "pvcreate /dev/xxx"
ansible -i hosts-list node -s -m shell -a "vgcreate vgdocker /dev/xxx"
ansible -i hosts-list node -s -m shell -a "lvcreate -L 100G -n lvdocker vgdocker"
ansible -i hosts-list node -s -m shell -a "mkfs -t xfs /dev/mapper/vgdocker-lvdocker"
ansible -i hosts-list node -s -m shell -a "mount /dev/mapper/vgdocker-lvdocker /var/lib/docker"
ansible -i hosts-list node -s -m shell -a "echo '/dev/mapper/vgdocker-lvdocker /var/lib/docker xfs defaults 0 0' >> /etc/fstab" 
ansible -i hosts-list node -s -m shell -a "service docker restart"
```
## 三. Install Openshift
```
ansible-playbook -i hosts openshift-ansible/playbooks/byo/config.yml
```
## 四. Install Brokers-Server
```
docker run -d -p 8443:8443 --name "openshift-origin" \
 --privileged --net=host \
-v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys:ro -v /var/lib/docker:/var/lib/docker:rw \
registry.dataos.io/openshift/ldp-origin:v1.1.6-ldp0.4.19 start

```
