# df-deploy
#内网环境OCDF部署
# 一.Planning
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
