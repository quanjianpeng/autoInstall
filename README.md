### 树懒·闪电
**欢迎加入树懒·闪电计划**  
**忘掉那些冗繁复杂的安装部署文档吧，自动化安装脚本才是你需要的！**

通过编辑linux或windows脚本，自动化部署安装所需软件，并优化相关配置，实现目标：

1.  一键安装部署服务器端生产环境，包括集群与分布式部署，并进行相关优化配置，如mysq主从，Apache和tomcat负载均衡……
2.  一键安装个人PC开发环境， 保证统一的、标准的开发环境，包括：java web开发，java rcp开发，C#客户端开发，Android开发，iOS开发……
3.  批量升级维护服务器软件
4.  定制解决方案，如高性能web服务器部署：jdk + 负载均衡 + 缓存服务器 + 数据库集群 + 静态资源服务……

---
举个栗子：mysql-5.6.30单机安装脚本（ install-mysql.sh ）

```
#!/bin/bash
#判断是否为超级管理员
if [ $UID -ne 0 ];
then
    echo "Please run with super user"
    exit 1
fi

#设置环境变量
#PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
#export PATH

## 读取配置文件
. ../config/my-config.sh
## 解压文件
mkdir $MYSQL_INSTALL_DIR
tar -zxvf $MYSQL_INSTALLED -C $MYSQL_INSTALL_DIR
#切换到安装目录
cd $MYSQL_INSTALL_DIR
#创建软连接（快捷方式）
ln -s $MYSQL_INSTALL_VERSION $MYSQL_VERSION
#切换当前目录
cd $MYSQL_VERSION
#新增用户组和用户
groupadd mysql
useradd -g mysql mysql
#改变当前目录的拥有者和用户组
chown -R mysql .
chgrp -R mysql .
#回到上级目录
cd ..
#复制服务文件到系统启动目录，并修改该权限
cp -f $MYSQL_INID /etc/init.d/mysqld
chmod a+x /etc/init.d/mysqld
#复制控制文件到系统默认路径
cp -f $MYSQL_CNF /etc/my.cnf
#创建快捷方式到用户环境变量，保证mysql客户端可以快速使用
ln -s $MYSQL_INSTALL_DIR/$MYSQL_VERSION/bin/mysql /usr/bin
#启动服务
service mysqld start
## 添加防火墙规则
sh $HOME_INSTALL/change-iptables.sh $MYSQL_PORT
## 初始化数据库
mysql -P $MYSQL_PORT –u root –p -D mysql < $HOME_INSTALL/init-mysql.sql

```

---

以上脚本保存后，只需要执行./install-mysql.sh（这里还依赖了两个通用的脚本，一个是全局环境变量配置脚本my-config.sh，一个是快速修改防火墙的脚本），即可快速安装mysql数据库。当然这个脚本还有很多不足之处，它没有定时备份计划，也没有定期删除日志计划，还需要不断去完善。  
通过多个脚本组合，就可以形成一个整体解决方案，快速安装部署服务器生产环境，乃至分布式集群化！！！  

请看**hadoop-2.4.0**家族自动化安装脚本
```
#!/bin/bash
## 说明：Hadoop-2.4.0家族环境自动化部署安装脚本
## 作者：yaodh
## 博客：http://www.cnblogs.com/lucky110100
## 码云：http://git.oschina.net/signup?inviter=lucky110100
set -e
## 检查输入参数
if [ $# -lt 3 ]; then
    echo "$0 错误：请传入参数，(1、类型:name/node;2、机器名：name00/node01;3、机器IP)！"
	echo "例如：$0 name name00 127.0.0.1"
	exit 2
fi

echo "参数确认： 类型：$1   机器名： $2    机器IP：$3"

read -p "继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	echo '					开始安装……'
else 
	echo '					退出安装！！！'
	exit 2
fi

if [ ! -f 'hadoop.info'  ];then
	touch hadoop.info
fi
#写入集群信息——很重要
sed -i "/$2/d" hadoop.info
echo "$1	$2	$3" >> hadoop.info

URL_SHELL=http://172.16.11.10:9300

read -p "SSH链接慢？(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "sed -i 's@#UseDNS yes@UseDNS no@g' /etc/ssh/sshd_config;sed -i 's@#   GSSAPIAuthentication no@GSSAPIAuthentication no@g' /etc/ssh/ssh_config;service sshd restart"
else 
	echo '					跳过……'
fi

read -p "授权公钥……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	wget $URL_SHELL/common/auth-key.sh;chmod a+x auth-key.sh;./auth-key.sh $1 $3
else 
	echo '					跳过……'
fi

read -p "下载脚本……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	##下载脚本，并删除已rpm安装的软件
	ssh root@$3 "mkdir -p /home/install"
	## 下载脚本
	#ssh root@$3 "cd /home/install;wget -N -r -nH -np -k -L -p $URL_SHELL" #层级下载有点问题
	## 下载脚本包
	ssh root@$3 "cd /home/install;rm -f linux.zip;wget $URL_SHELL/linux.zip"
	## 解压并提权
	ssh root@$3 "cd /home/install;unzip -o linux.zip;chmod a+x /home/install -R"
else 
	echo '					跳过……'
fi

read -p "删除已rpm安装的软件……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	## 删除rpm安装的软件
	ssh root@$3 "cd /home/install/common;./uninstall-rpm.sh"
else 
	echo '					跳过……'
fi

read -p "安装java……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/java;./installed-jdk.sh"
else 
	echo '					跳过……'
fi

read -p "安装mysql……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/mysql;./installed-mysql.sh 1 single"
else 
	echo '					跳过……'
fi

read -p "安装hadoop……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/hadoop;./hadoop-single.sh $1 $2 $3"
else
	echo '					跳过……'
fi

read -p "安装hive……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/hadoop;./hive-single.sh"
else
	echo '					跳过……'
fi

read -p "安装zookeeper……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	echo '					未完成……'
else 
	echo '					跳过……'
fi

read -p "安装hbase……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/hadoop;./hadoop-single.sh $1 $2 $3"
else 
	echo '					跳过……'
fi

read -p "设置环境变量……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	ssh root@$3 "cd /home/install/config;./my-profile.sh;source /etc/profile"
	ssh root@$3 "source /etc/profile"
else 
	echo '					跳过……'
fi

read -p "立即加入Hadoop集群……继续/跳过(Y/N):" isY
if [ "${isY}" == "y" ] || [ "${isY}" == "Y" ];then
	./hadoop-cluster.sh
else 
	echo '					跳过……'
fi
```

升级版，菜单交互安装
```
        12. 安装zookeeper
        13. 集群zookeeper
        14. 安装hbase
        15. 集群hbase
        16. 设置环境变量
请输入数字选择：1
********************************************************
新节点： 类型：node   机器名： node11705    机器IP：172.16.117.5
********************************************************
********************************************************
hadoop集群信息：
node    node11703       172.16.117.3
node    node11704       172.16.117.4
node    node11903       172.16.119.3
name    name00  172.16.119.0
node    node11901       172.16.119.1
node    node11902       172.16.119.2
********************************************************
********************************************************
zookeeper集群信息：
name    name00  172.16.119.0
node    node11901       172.16.119.1
node    node11902       172.16.119.2
********************************************************
********************************************************
hbase集群信息：
name    name00  172.16.119.0
node    node11901       172.16.119.1
node    node11902       172.16.119.2
********************************************************

                准备安装
        0. 退出
        1. 检查集群信息
        2. 写入集群信息
        3. 解决SSH链接慢，并自动添加到know_hosts
        4. 授权公钥
        5. 下载脚本
        6. 删除rpm安装的软件
        7. 安装jdk
        8. 安装mysql
        9. 安装hadoop
        10. 集群Hadoop
        11. 安装hive
        12. 安装zookeeper
        13. 集群zookeeper
        14. 安装hbase
        15. 集群hbase
        16. 设置环境变量
请输入数字选择：0
```

---
###### 最后，有两个要求： 

###### 1、安装过程中需要修改或替换系统中原有的文件前，要进行备份，这个非常非常非常重要！！！ 
###### 2、既然有安装那么就要有卸载脚本，一旦安装过程中出现问题，我们可以把添加修改过的文件还原，重新安装，坚持谁污染谁治理原则！