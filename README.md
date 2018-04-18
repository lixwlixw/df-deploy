# df-deploy
#内网环境OCDF部署
# 一.Planning
masterx3  
etcdx3  
lbx1  
nodex3  
origin-1.2x1  
service-brokers-etcdx1  
glusterfsx2  
heketix1  
nfsx1  
yum mirrorx1  
docker mirrorx1  
dnsx1  


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
