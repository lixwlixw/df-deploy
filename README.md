# df-deploy
#内网环境OCDF部署
# 一.Planning
master*3 
etcd*3 
lb*1 
node*N  
origin-1.2*1 
service-brokers etcd *1
glusterfs*2 
heketi*1 
nfs*1 
yum mirror*1  
docker mirror*1
dns*1 


# 二. Preparation
    
  1 搭建yum 仓库
     安装httpd服务
     把准备的yum源放到/var/www/html/ 下
     createrepo /var/www/html/xxx
     service httpd start    
     
  2 给机器配置本地yum源
     
     
  3 配置机器互信
     控制机生成key文件 ssh-keygen -t rsa
     配置控制机和其他机器互信  ssh-copy-id root@IP

  4 通过ansible 给所有机器配置yum客户端文件
