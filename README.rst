

### Ansible-playbook-rhcs 配置文档

文档背景：


    本文档根据rhcs部署环境编写

文档假设环境为：


    hosts ：根据《RHEL操作系统安装规范》系统标准化处理过的VM模板安装的VM
    number：2台（rhcsmaster、rhcsslaver）
    os    ：rhel6.4
    server：任意装有ansible的rhel操作系统，截止此文档编写时（2016-03-25），ansible版本为1.9.4

部署流程：


    1）假设ansible server 及 2台rhcs hosts已准备就绪
    2）将准备好的ansible-playbook-rhcs.tgz按照文档解压后备用
    3）配置2台rhcs hosts的网络：
  	3.1）需要提供主机的网络规划信息：
	    心跳网1、心跳网2、业务网1、业务网2、管理网等5种网络的ipaddress、gateway
      	业务网1、业务网2的VIP（跟业务网1、业务网2分别同网段）
   	    心跳网1、心跳网2、业务网1、业务网2的dns1、dns2
	    
       3.2）需要动手配置的文件：（具体操作参考文档）
           rhcs/group_vars/{all,rhcsmaster,rhcsslaver}
           rhcs/hosts

	3.3)需要提供：yum repo server
                
    4）使用 #ansible-playbook -i hosts site.yml 命令开始自动化部署
	自动化部署的流程：
	    自动化部署ipaddress（配置网络）（resource：rhcs/roles/ipaddress）
	    自动化部署common（通用配置）（resource：rhcs/roles/common）
	    自动化部署rhcs（rhcs相关配置）（resource：rhcs/roles/rhcs）
	    自动化部署mysql(以模块化的方式添加服务到rhcs中)（resource：rhcs/roles/mysql）
	    自动化部署zabbix（待完成）（添加zabbix客户端）            
    配置完毕


大批量rhcs环境部署方案：

    将需要部署的主机以主机对（rhcsmaster、rhcsslaver）的形式分配好ip、vip、网关、dns
    使用shell或python脚本调用ansbile-playbook-rhcs进行自动化部署

进行网络规划



1、环境准备：



	1）3台主机

	ansible-server

	rhcsmaster （5网卡）

	rhcsslaver （5网卡）


2、网络规划：

	

    rhcsmaster （至少三块网卡，如果网卡数不够，心跳网可与业务网复用）

	99.12.137.109   DBASMY-DB41KF-HB1    eth0  //心跳网1

	99.12.138.171   DBASMY-DB41KF-HB2    eth1  //心跳网2

	99.12.135.5     DBASMY-DB41KF-APP1   eth2  //业务网1

	99.12.136.24    DBASMY-DB41KF-APP2   eth3  //业务网2

	99.12.139.54    DBASMY-DB41KF        eth4  //管理网 

	

    rhcsslaver

	99.12.137.110   DBASMY-DB42KF-HB1	eth0	//心跳网1

	99.12.138.178   DBASMY-DB42KF-HB2	eth1	//心跳网2

	99.12.135.7     DBASMY-DB42KF-APP1	eth2	//业务网1

	99.12.136.29    DBASMY-DB42KF-APP2	eth3	//业务网2

	99.12.139.53    DBASMY-DB42KF       eth4    //管理网

	

    网关

	eth0心跳网1网关：99.12.137.253

	eth1心跳网2网关：99.12.138.253

	eth2业务网1网关；99.12.135.253

	eth3业务网2网关：99.12.136.253

	eth4管理网网关：99.12.139.253

	

    VIP

 	99.12.135.70    VIP for app1		  // for 业务网1

	99.12.136.30    VIP for app2          // for 业务网2

	

    DNS

	DNS1=99.1.34.23

	DNS2=99.1.34.25

	

3、将playbook解压到/opt目录（ansible-server端）


	[root@ansible-server rhcs]# tar zxvf ansible-playbook-rhcs.tgz -C /opt

	
	ansible-playbook-rhcs的目录结构

	├── group_vars					//变量定义目录			
	│   ├── all					    //全局变量文件			
	│   ├── rhcsmaster				//rhcsmaster专属变量文件			
	│   ├── rhcsservers
	│   └── rhcsslaver				//rhcsslaver专属变量文件
	├── hosts					
	├── iscsi.yml					   //hosts主机配置文件
	├── library					   //自定义ansible模块目录					
	│   ├── insert_file
	│   └── read_file
	├── LICENSE.md
	├── parted.yml
	├── README.md
	├── roles					  //ansible plays 目录
	│   ├── common					//通用配置play
	│   │   ├── handlers
	│   │   │   └── main.yml
	│   │   ├── tasks
	│   │   │   └── main.yml
	│   │   └── templates
	│   │       ├── etc_hosts.j2			//配置/etc/hosts文件的模板
	│   │       └── rhel6.repo.j2			//配置所需yum源的模板
	│   ├── ipaddress				//ipaddres配置play
	│   │   ├── handlers
	│   │   │   └── main.yml
	│   │   ├── tasks
	│   │   │   └── main.yml
	│   │   └── templates
	│   │       ├── ifcfg-eth0.j2
	│   │       ├── ifcfg-eth1.j2
	│   │       ├── ifcfg-eth2.j2
	│   │       ├── ifcfg-eth3.j2
	│   │       ├── route-eth2.j2
	│   │       ├── route-eth3.j2
	│   │       ├── route.sh.j2
	│   │       ├── rule-eth2.j2
	│   │       ├── rule-eth3.j2
	│   │       └── static-routes.j2
	│   ├── mysql					//mysql服务配置play 
	│   │   ├── defaults
	│   │   ├── handlers
	│   │   │   └── main.yml
	│   │   ├── tasks
	│   │   │   └── main.yml
	│   │   ├── templates
	│   │   │   ├── mysql_resource.j2
	│   │   │   └── mysql_server.j2
	│   │   └── vars
	│   │       └── main.yml
	│   ├── rhcs					//rhcs服务配置play
	│   │   ├── handlers
	│   │   │   └── main.yml
	│   │   ├── tasks
	│   │   │   └── main.yml
	│   │   └── templates
	│   │       ├── checkqdisk.py.j2		
	│   │       ├── cluster.conf.j2		//rhcs核心配置文件
	│   │       ├── initiatorname.iscsi.j2 	
	│   │       ├── qdisk_master.sh.j2		
	│   │       ├── qdisk_server.sh.j2		
	│   │       ├── qdisk_vip.sh.j2		
	│   │       ├── route.sh.j2			
	│   │       ├── test_in_rhcs.j2		
	│   │       ├── vgscan.py.j2			
	│   │       └── vip_server.sh.j2		
	│   └── zabbix					//zabbix服务配置play			
	│       ├── handlers
	│       │   └── main.yml
	│       ├── tasks
	│       │   └── main.yml
	│       └── templates
	└── site.yml					//综合play

	



4、配置ansible管理两台rhcs机器（ansible-server端）



	[root@ansible-server rhcs]# vim rhcs/hosts


	[rhcsmaster]

	rhcsmaster ansible_ssh_host=99.12.139.54 (rhcsmaster管理网eth4 对应的ip)

	

	[rhcsslaver]

	rhcsslaver ansible_ssh_host=99.12.139.53 (rhcsslaver管理网eth4 对应的ip)



	
 	拷贝ssh公钥


	1）生成root用户的公钥（直接回车）

	[root@ansible-server rhcs]# ssh-keygen 


	Generating public/private rsa key pair.

	Enter file in which to save the key (/root/.ssh/id_rsa):  //回车

	Enter passphrase (empty for no passphrase):  //回车

	Enter same passphrase again: //回车

	Your identification has been saved in /root/.ssh/id_rsa.

	Your public key has been saved in /root/.ssh/id_rsa.pub.

	The key fingerprint is:

	9b:68:fa:45:96:42:e2:3e:3b:08:ea:16:d0:6d:8e:5f root@ansible.example.com

	The key's randomart image is:

	+--[ RSA 2048]----+

	|                 |

	|                 |

	| . .. .          |

	|. ..oo   .       |

	|.  +. . S        |

	|..... E= o       |

	|...oo.o +        |

	|... o= .         |

	|o.  oo.          |

	+-----------------+




	2）拷贝公钥到rhcsserver和rhcsslaver

	[root@ansible-server rhcs]# ssh-copy-id root@99.12.139.54 

	  root@99.12.139.54's password:    // your password



	[root@ansible-server rhcs]# ssh-copy-id root@99.12.139.53 

	  root@99.12.139.53's password:    // your password



	测试是否成功：


	[root@ansible-server rhcs]# ssh root@99.12.139.54

     Last login: Fri Mar 18 14:28:03 2016 from 99.12.156.9


    [root@ansible-server rhcs]# ssh root@99.12.139.53

	Last login: Fri Mar 18 14:28:10 2016 from 99.12.156.9


	不再用输入密码直接可以登录

	
	使用ansible ping模块测试主机的管理网通不通


	[root@ansible-server rhcs]# ansible all -m ping -i hosts


	rhcs1 | success >> {

		"changed": false, 

		"ping": "pong"

	}



	rhcs2 | success >> {

		"changed": false, 

		"ping": "pong"

	}

	

       说明rhcsmaster和rhcsslaver可以被ansible管理了


	

5、需要手动配置的文件
	
	1）配置全局参数：（ansible-server端，file：rhcs/group_vars/all）
        

	[root@ansible rhcs]# cat group_vars/all 

	#

	# Set IP Address	 //需要手动添加

	#

	network_root: /etc/sysconfig/network-scripts



	eth0_netmask: 255.255.255.0

	eth0_gateway: 99.12.137.253

	eth0_cidr: 99.12.137.0/24



	eth1_netmask: 255.255.255.0

	eth1_gateway: 99.12.138.253

	eth1_cidr: 99.12.138./24



	eth2_netmask: 255.255.255.0

	eth2_gateway: 99.12.135.253

	eth2_cidr: 99.12.135.0/24



	eth3_netmask: 255.255.255.0

	eth3_gateway: 99.12.136.253

	eth3_cidr: 99.12.136.0/24



	vip_for_app1: 99.12.135.70

	vip_for_app2: 99.12.136.30



	dns1: 99.1.34.23

	dns2: 99.1.34.25



	vcenter_server_ip: 99.12.131.102

	#
	# Set yum repos.           //需要手动修改	    
	#
	yum_server_path: http://99.1.14.11/rhel64

	#
	# Set Client /etc/hosts   //需要手动修改
	#
	heartbeat1_master_ip: 99.12.137.109
	heartbeat1_slaver_ip: 99.12.137.110

	heartbeat2_master_ip: 99.12.138.171
	heartbeat2_slaver_ip: 99.12.138.178

	heartbeat1_master_hostname: DBASMY-DB41KF-HB1
	heartbeat1_slaver_hostname: DBASMY-DB42KF-HB1

	heartbeat2_master_hostname: DBASMY-DB41KF-HB2
	heartbeat2_slaver_hostname: DBASMY-DB42KF-HB2

	#
	# Set heuristic_shell_scripts		// 默认配置，不需要手动修改
	#
	heuristic_shell_scripts: [
        	qdisk_master.sh,
        	qdisk_server.sh,
        	qdisk_vip.sh,
        	vip_server.sh
	]

	#
	# Set qdisk.			// 默认配置，其他客户需要手动指定
	#
	dev_name: sdc
	dev_part_name: sdc1
	qdisk_label: mysqlqdisk
	disk_size: 100% 				
	
	#
	# Set cluster /etc/cluster/cluster.conf	//默认配置，其他客户需要手动指定
	#
	config_version: 22
	heartbeat1_master_priority: 1
	heartbeat1_slaver_priority: 3

	ricci_password: redhat

	cluster_name: mycluster
	cluster_conf: /etc/cluster/cluster.conf
	cluster_fence_name: MySQL-Fence
	cluster_fence_method: vmware_soap

	master_vm_name: DBASMY-DB41KF
	master_vm_uuid: "{{hostvars['rhcs1'].ansible_product_uuid}}"

	slaver_vm_name: DBASMY-DB42KF
	slaver_vm_uuid: "{{hostvars['rhcs2'].ansible_product_uuid}}"

	altmulticast_addr: 224.192.204.10
	quorum_dev_poll: 30000

	fence_agent_ipaddr: 99.12.131.102
	fence_agent_login_user: vmrhcs01
	fence_agent_login_password: '@rhcs1215%'

	failoverdomain_name: MySQL-Domian

	mysql_start_script_path: /etc/rc.d/init.d/mysqld_new

	cman_timeout: 30000

	cluster_vip_log: /var/log/cluster/vipinfo.log
	
	#
	# mysql dependencies		// 添加mysql服务时用到的参数，如果不安装mysql服务，可以注释掉
	#
	mysql_dep_list: [
 	   expect,
 	   iscsi-initiator-utils,
 	   MySQL-python,
 	   perl-DBD-MySQL,
 	   perl-Time-HiRes,
 	   php,
 	   php-mysql,
  	  perl-DBD-MySQL,
  	  pcre-devel,
  	  zlib-devel,
  	  openssl-devel,
  	  gcc,
  	  java-1.7.0-openjdk.x86_64
	]

	#
	# Server for mysql		// 添加mysql服务时用到的参数，如果不安装mysql服务，可以注释掉
	#
	mysql_server_name: MySQL-Server
	

		

	2）配置rhcsmaster专用参数（ansible-server端，file：group_vars/rhcsmaster）

	
	---
	#
	# Set IP Address.	//需要手动修改
	#
	eth0_ip: 99.12.137.109
	eth1_ip: 99.12.138.171
	eth2_ip: 99.12.135.5
	eth3_ip: 99.12.136.24

	#
	# Create qdisk.
	#
	createqdisk: True

	#
	# Set HA-LVM /etc/lvm/lvm.conf
	#
	hb_name: "{{heartbeat1_master_hostname}}"


	


	3)配置rhcsslaver专用参数（ansible-server端，file：group_vars/rhcsslaver）

	---
	#
	# Set IP Address.	//需要手动修改
	#	
	eth0_ip: 99.12.137.110
	eth1_ip: 99.12.138.178
	eth2_ip: 99.12.135.7
	eth3_ip: 99.12.136.29

	#
	# Create qdisk.
	#
	createqdisk: False

	#
	# Set HA-LVM /etc/lvm/lvm.conf
	#
	hb_name: "{{heartbeat1_slaver_hostname}}"


6、 使用ansible-playbook-rhcs自动化部署rhcs服务（ansible-server端)
	
	1）只部署rhcs服务
	[root@ansible-server rhcs]# cd /opt/rhcs
	[root@ansible-server rhcs]# ansible-playbook -i hosts site.yml --skip-tags=mysql

	2) 同时部署rhcs和mysql
	[root@ansible-server rhcs]# cd /opt/rhcs
	[root@ansible-server rhcs]# ansible-playbook -i hosts site.yml





