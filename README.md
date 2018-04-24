# DF-DEPLOY-DOCS.
## 一. Planning.
```
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
```
## 二. Preparation.
    
1. Install Yum Repository.  
```
tar -xf yumrepo.tar -C /var/www/html/
createrepo /var/www/html/base/
createrepo /var/www/html/xxxx/
service httpd start
```     
2. Ensuring Host Access.
```
ssh-keygen -t rsa

for i in master1 \
 master2 \
 master3 \
 xxxxxx ; \
 do ssh-copy-id -i $i; \
 done
```     
3. Use Ansible Configuration Host.
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
4. Install Configuration Docker Repository.
```
docker run -d -p 5000:5000 registry:2

tar -xf docker-images.tar

cd docker-images/

for i in `ll|awk '{print $9}'`; do docker load < $i; done

docker images |grep"dataos.io"|awk '{print "docker tag "$3""$1":"$2}'| \
  sed-e s/registry.new.dataos.io/10.1.1.x:5000/| \
 xargs -i bash -c "{}"
 
docker images |grep "10.1.1.x:5000"| \
  awk'{print "docker push "$1":"$2}'| \
 xargs -i bash -c "{}"
```
5. Configuration Insecure-Registry.
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
6. Configuration Docker Storage.
```
ansible -i hosts-list node -s -m shell -a "pvcreate /dev/xxx"
ansible -i hosts-list node -s -m shell -a "vgcreate vgdocker /dev/xxx"
ansible -i hosts-list node -s -m shell -a "lvcreate -L 100G -n lvdocker vgdocker"
ansible -i hosts-list node -s -m shell -a "mkfs -t xfs /dev/mapper/vgdocker-lvdocker"
ansible -i hosts-list node -s -m shell -a "mount /dev/mapper/vgdocker-lvdocker /var/lib/docker"
ansible -i hosts-list node -s -m shell -a "echo '/dev/mapper/vgdocker-lvdocker /var/lib/docker xfs defaults 0 0' >> /etc/fstab" 
ansible -i hosts-list node -s -m shell -a "service docker restart"
```
## 三. Install Openshift.
openshift-ansible file address https://github.com/openshift/openshift-ansible/tree/release-3.6

```
ansible-playbook -i hosts openshift-ansible/playbooks/byo/config.yml
```
hosts file example
```
[OSEv3:children]
masters
etcd
nodes
lb
glusterfs

[OSEv3:vars]
#login vm
ansible_ssh_user=ocdf
ansible_become=true
deployment_type=origin

#openshift images url
osm_image=10.1.1.x:5000/openshift/origin
osn_image=10.1.1.x:5000/openshift/node
osn_ovs_image=10.1.1.x:5000/openshift/openvswitch
openshift_image_tag='v3.6.0'
oreg_url=10.1.1.x:5000/openshift/origin-${component}:${version}

#key time
openshift_hosted_registry_cert_expire_days = 1825
openshift_ca_cert_expire_days = 1825
openshift_node_cert_expire_days = 1825
openshift_master_cert_expire_days = 1825
etcd_ca_default_days = 1825

#cluster
openshift_master_cluster_method=native
openshift_master_cluster_hostname=10.1.1.x
openshift_master_cluster_public_hostname=10.1.1.x
openshift_master_api_port=8443
openshift_master_console_port=8443
osm_cluster_network_cidr=172.26.0.0/16
openshift_portal_net=172.25.0.0/16
osm_host_subnet_length=8
enable_excluders=false
#openshift_clock_enabled=true

#etcd image
osm_etcd_image=10.1.1.x:5000/openshift/etcd

#check
#openshift_disable_check=memory_availability,disk_availability,docker_storage,docker_image_availability,docker_storage_driver,package_version,package_availability,package_update
openshift_disable_check=memory_availability,disk_availability,docker_storage,docker_image_availability,package_version
openshift_node_kubelet_args={'pods-per-core': ['10'], 'max-pods': ['250'], 'image-gc-high-threshold': ['80'], 'image-gc-low-threshold': ['70']}
openshift_master_default_subdomain=xxx.xxx.com

#metrics
openshift_hosted_metrics_deploy=true
openshift_hosted_metrics_deployer_prefix=10.1.1.x:5000/openshift/origin-
openshift_hosted_metrics_deployer_version=v3.6.0

#logging
openshift_hosted_logging_deploy=true
openshift_hosted_logging_es_cluster_size=1
openshift_hosted_logging_deployer_prefix=10.1.1.x:5000/openshift/origin-
openshift_hosted_logging_deployer_version=v3.6.0
openshift_hosted_logging_master_public_url=https://10.1.1.x:8443
openshift_hosted_logging_hostname=xxx.xxx.com

#glusterfs enable
#glusterfs configuration
openshift_storage_glusterfs_namespace=glusterfsdb 
openshift_storage_glusterfs_name=storage
openshift_storage_glusterfs_image=10.1.1.x:5000/gluster/gluster-centos
openshift_storage_glusterfs_version=v3.6.0
openshift_storage_glusterfs_block_image=10.1.1.x:5000/gluster/glusterblock-provisioner
openshift_storage_glusterfs_block_version=v3.6.0
openshift_storage_glusterfs_heketi_image=10.1.1.x:5000/heketi/heketi
openshift_storage_glusterfs_heketi_version:v3.6.0

# host group
[masters]
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x

[etcd]
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x
10.1.1.x containerized=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x

[lb]
10.1.1.x openshift_hostname=10.1.1.x openshift_ip=10.1.1.x

[nodes]
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=false openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=false openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

[glusterfs]
10.1.1.x glusterfs_ip=10.1.1.x glusterfs_devices='[ "/dev/vdb" ]'
10.1.1.x glusterfs_ip=10.1.1.x glusterfs_devices='[ "/dev/vdb" ]'
10.1.1.x glusterfs_ip=10.1.1.x glusterfs_devices='[ "/dev/vdb" ]'

#[new_nodes]
#10.1.1.x containerized=true openshift_docker_options="--log-driver json-file --log-opt max-size=3M --log-opt max-file=3" openshift_schedulable=true openshift_hostname=10.1.1.x openshift_ip=10.1.1.x openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
```
## 四. Install Components.

1. Start The DataFoundryWeb.
```
oc new-project datafoundry
oc create -f datafoundry/datafoudrygitter.yaml
oc create -f datafoundry/datafoundrypayment.yaml
oc create -f datafoundry/datafoundryvolume.yaml
oc create -f datafoundry/datafoundryweb.yaml
```

2. Start The Service-Broker Container.
```
oc new-project service-brokers
oc create -f datafoundry/service-brokers.yaml
```      
3. Start The Etcd Service.
```
docker run -d -p 2380:2380 -p 2379:2379 \
 --name etcd 10.1.1.x:5000/coreetcd/etcd:v2.3.7 \
 -name etcd0 \
 -advertise-client-urls http://10.1.1.x:2379 \
 -listen-client-urls http://0.0.0.0:2379 \
 -initial-advertise-peer-urls http://10.1.1.x:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster etcd0=http://10.1.1.x:2380 \
 -initial-cluster-state new
```     
4. Configuration Etcd Login And Permissions.      
If you want to know etcd.sh. Please see https://github.com/lileitongxue/ETCD.git
```
yum -y install etcd

etcdctl user add username << EOF
password
EOF

etcdctl auth enable

etcdctl -u username:password role revoke guest --path '/*' -readwrite

sh -x etcd.sh
```       
5. Start The Origin1.2 And Create ServiceBrokers In OpenShift.
```
docker run -d -p 8443:8443 --name "openshift-origin" \
 --privileged --net=host \
-v /:/rootfs:ro -v /var/run:/var/run:rw -v /sys:/sys:ro -v /var/lib/docker:/var/lib/docker:rw \
registry.dataos.io/openshift/ldp-origin:v1.1.6-ldp0.4.19 start

docker exec -it openshift-origin bash

oc new-servicebroker etcd --username=xxx --password=xxx --url=http://servicebroker.xxx.com
```       
6. Install DNS.
```
yum -y install dnsmasq

echo "address=/xxx.com/10.1.1.x" > /etc/dnsmasq.d/address.conf

service dnsmasq start
```
