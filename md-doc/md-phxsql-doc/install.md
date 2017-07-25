#环境
CentOS release 6.6
/opt/rh/devtoolset-3/root/usr/bin/g++ --version

g++ (GCC) 4.9.1 20140922 (Red Hat 4.9.1-10)

install  binutils-2.27.tar.gz
./configure;make ;make install

cp /usr/local/bin/ar /opt/rh/devtoolset-3/root/usr/bin/ar

cp /usr/local/bin/ranlib /opt/rh/devtoolset-3/root/usr/bin/ranlib

/opt/rh/devtoolset-3/root/usr/bin/ar --version
GNU ar (GNU Binutils) 2.27

yum install readline-devel.x86_64

#源码

* git clone --recursive https://github.com/tencent-wechat/phxsql.git


* git clone --recursive -bv0.8.0  https://github.com/tencent-wechat/phxsql.git

git clone --recursive https://github.com/tencent-wechat/phxpaxos.git

* cd third_party

* ./autoinstall.sh

* wget https://www.percona.com/downloads/Percona-Server-5.6/Percona-Server-5.6.31-77.0/source/tarball/percona-server-5.6.31-77.0.tar.gz

在公司只能网页上下载，然后再上传

* tar zxvf percona-server-5.6.31-77.0.tar.gz

* mv percona-server-5.6.31-77.0 percona

* cd /root/phxsql

* ./autoinstall.sh

* make;make install

然后把整个目录打包

* tar -czvf phxsql.tar.gz phxsql/* phxsql/.[!.]*

包含隐藏文件，scp到其他主机

* cd /root/phxsql/install_package
ls

#部署
编辑 etc_template/my.cnf

bind-address = 0.0.0.0
proxy_protocol_networks = 127.0.0.1

每台机器执行：

* mkdir -p /data/phxsql/

* cd /root/phxsql/install_package/tools

* cd /root/phxsql/tools

* python ./install.py -i 10.199.201.143 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001 

python ./install.py -i 10.199.201.144 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001 

python ./install.py -i 10.199.201.145 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001

python ./install.py -i 10.199.200.189 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001

//-f/tmp/data/使用默认的/data/

#停止
python kill.py        

#check
ps -ef | grep phxsqlproxy
ps -ef | grep mysql
ps -ef | grep phxbinlogsvr


启动正常

#两篇文章

http://www.zhanghaijun.com/post/968/

http://www.zhanghaijun.com/post/969/

#初始化

rm -rf /root/phxsql/install_package;rm -rf /data/phxsql/;mkdir -p /data/phxsql/

端口监听情况：

phxsqlproxy_phx 54321

phxbinlogsvr_ph 6000

mysqld 11111

/root/phxsql/install_package/sbin/mysqld

phxbinlogsvr_ph  17000

cd /root/phxsql/sbin

./phxbinlogsvr_tools_phxrpc -f InitBinlogSvrMaster -h 10.199.201.143,10.199.201.144,10.199.201.145 -p 17000

~~~
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
echo 1 > /proc/sys/net/ipv6/conf/default/disable_ipv6



get master  expire time 0
get master  expire time 0
get master  expire time 0
init master 10.199.201.143 done, start to add member
get master  expire time 0
waiting master to be started
get master fail ret -1003
waiting master to be started
get master 10.199.201.143 expire time 1488425224
master started, ip 10.199.201.143
add ip 10.199.201.144 to master done
add ip 10.199.201.145 to master done

~~~


#杀

cd /root/phxsql/tools/

python ./kill.py

rm -rf /root/phxsql;rm -rf /data/phxsql

cd /root;tar zxvf phxsql-0.8.5.tar.gz 

cd /root/phxsql/tools/

 mkdir -p /data/phxsql
 
 rm -rf /home/zhangwusheng/data
 mkdir -p /home/zhangwusheng/data
 chmod a+rw /home/zhangwusheng/data
 
  cd /home/zhangwusheng/phxsql/tools/
  
  
  
#去掉mysql
  vi ./binary_installer.py:39
  
  ./directory_operator.py:27
  

python ./install.py -i 10.199.201.143 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001  -f /home/zhangwusheng/data

python ./install.py -i 10.199.201.144 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001 -f /home/zhangwusheng/data

python ./install.py -i 10.199.201.145 -p 54321 -g 6000 -y 11111 -P 17000 -a 8001 -f /home/zhangwusheng/data

/root/phxsql/percona.src/bin/mysql  -uroot -p -S/data/phxsql/percona.workspace/tmp/percona.sock


/root/phxsql/percona.src/bin/mysql -uroot -p -h10.199.201.143 -P11111

54321
/home/zhangwusheng/phxsql/percona.src/bin/mysql -uroot -p -h10.199.201.143 -P11111


./phxbinlogsvr_tools_phxrpc -f InitBinlogSvrMaster -h 10.199.201.143,10.199.201.144,10.199.201.145 -p 17000



/home/zhangwusheng/phxsql/percona.src/bin/mysql  -uroot -h 10.199.201.143  -P54321

use test_phxsql;

create table benchmark1(col1 int,col2 varchar(20),col3 int,col4 varchar(30) );



bash ./test_phxsql.sh 54321 10.199.201.144



sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=5 --max-requests=10000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql prepare

sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=100 --num-threads=5 --max-requests=1000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql run


sysbench --oltp-tables-count=10 --oltp-table-size=1000000 --num-threads=500 --max-requests=100000 --report-interval=1 --max-time=200

--mysql-host=

#hao 

/home/paxos/3rd/percona-5.6.31/bin/mysqld --defaults-file=/home/paxos/3rd/percona-5.6.31/my.cnf


./phxbinlogsvr_tools_phxrpc -f SetMySqlAdminInfo -h 10.199.201.143 -p 17000 -u root -d ""  -U admin -D admin

* ./phxbinlogsvr_tools_phxrpc -f GetMasterInfoFromGlobal -h 10.199.201.145 -p 17000



./phxbinlogsvr_tools_phxrpc -f GetMemberList -h 10.199.201.143 -p 17000

./phxbinlogsvr_tools_phxrpc -f SetMySqlAdminInfo -h 10.199.201.143  -p 17000 -u root -d "" -U zws -D zws  



/root/phxsql/percona.src/bin/mysql -uroot -S/data/phxsql/percona.workspace/tmp/percona.sock -e "select * from mysql.user\G"



cd /root/phxsql/tools
bash ./test_phxsql.sh 54321 10.199.201.143 10.199.201.144 10.199.201.145

/root/phxsql/percona.src/bin/mysql -uroot -h"10.199.201.145" -P54321

/root/phxsql/percona.src/bin/mysql -uroot -h 10.199.201.143 -P54322

/root/phxsql/percona.src/bin/mysqldump --set-gtid-purged=off -h 10.199.201.143 -P11111 -u root --all-databases >/home/all.sql

./phxbinlogsvr_tools_phxrpc -f AddMember -h 10.199.201.143 -p 17000 -m 10.199.201.145



* ./phxbinlogsvr_tools_phxrpc -f RemoveMember -h 10.199.201.143 -p 17000 -m 10.199.200.145 

* 上面RemoveMember这个出错啦！！


* ./phxbinlogsvr_tools -f SetMySqlAdminInfo -h 10.199.201.143 -p 17000 -u root -d "" -U root -D root
 
 这个也失败啦！！



./phxbinlogsvr_tools -f SetMySqlReplicaInfo -h 10.199.201.143 -p 17000 -u root -d "" -U replica -D replica


#半同步

ref <http://www.tuicool.com/articles/3aiaiai> 

首先修改slave的my.cnf的server_id

scripts/mysql_install_db  --defaults-file=/home/paxos/3rd/percona-5.6.31/my.cnf  --user=mysql

/home/paxos/3rd/percona-5.6.31/bin/mysqld --defaults-file=/home/paxos/3rd/percona-5.6.31/my.cnf


select @@have_dynamic_loading;  
master:
install plugin rpl_semi_sync_master soname 'semisync_master.so';

slave:
install plugin rpl_semi_sync_slave soname 'semisync_slave.so';  

show plugins;  


master:
set global rpl_semi_sync_master_enabled=1;  
set global rpl_semi_sync_master_timeout=1000; 
show global variables like '%rpl%';  


slave:

set global rpl_semi_sync_slave_enabled=1;  


grant replication slave on *.* to 'semirepl'@'10.199.201.145' identified by 'semirepl';
flush privileges;
show master status;

* slave:

change master to
     master_host='10.199.201.143',
    master_port=4444,
    master_user='semirepl',
    master_password='semirepl',
    master_log_file='bin-file.000035',    //刚刚Master那个File字段
    master_log_pos=407;     

change master to master_host='10.199.201.143',master_port =4444,master_user='semirepl',master_password='semirepl',master_auto_position=1;

slave:
set global rpl_semi_sync_slave_enabled=1;  
Show status like '%semi%'; 


master:
Show status like '%semi%'; 

#Percona

cd /root/percona-server-5.6.31

只要加了--user=mysql就会出错。
不加就可以运行
scripts/mysql_install_db  --defaults-file=/root/percona-server-5.6.31/my.cnf --basedir=/root/percona-server-5.6.31  --user=mysql

scripts/mysql_install_db --basedir=/root/percona-server-5.6.31 --defaults-file=/root/percona-server-5.6.31/my.cnf


scripts/mysql_install_db --basedir=/root/percona-server-5.6.31 --user=mysql


/root/percona-server-5.6.31/bin/mysqld_safe --defaults-file=/root/percona-server-5.6.31/my.cnf


cat my.cnf.phx|sed -e 's:/data/phxsql/percona.workspace:/root/percona-server-5.6.31:g' -e 's:/home/root/root:/root:g' -e 's:/root/phxsql/lib:/root/percona-server-5.6.31/plugin:g'> my.cnf.new

mkdir -p /root/percona-server-5.6.31/tmp
mkdir -p /root/percona-server-5.6.31/innodb
mkdir -p /root/percona-server-5.6.31/binlog
mkdir -p /root/percona-server-5.6.31/plugin

#BenchMark
1. 测试1
 	* 命令
> sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=5 --max-requests=100000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql prepare/run


   * 结果
    
OLTP test statistics:
    queries performed:
        read:                            386120
        write:                           137900
        other:                           55160
        total:                           579180
    transactions:                        27580  (137.73 per sec.)
    deadlocks:                           0      (0.00 per sec.)
    read/write requests:                 524020 (2616.87 per sec.)
    other operations:                    55160  (275.46 per sec.)

General statistics:
    total time:                          200.2472s
    total number of events:              27580
    total time taken by event execution: 1000.7764
    response time:
         min:                                  8.75ms
         avg:                                 36.29ms
         max:                                364.09ms
         approx.  95 percentile:              84.57ms

Threads fairness:
    events (avg/stddev):           5516.0000/4.69
    execution time (avg/stddev):   200.1553/0.05




sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=50 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql run


OLTP test statistics:
    queries performed:
        read:                            1165738
        write:                           416319
        other:                           166528
        total:                           1748585
    transactions:                        83261  (416.08 per sec.)
    deadlocks:                           6      (0.03 per sec.)
    read/write requests:                 1582057 (7905.94 per sec.)
    other operations:                    166528 (832.18 per sec.)

General statistics:
    total time:                          200.1099s
    total number of events:              83261
    total time taken by event execution: 9999.3496
    response time:
         min:                                 18.93ms
         avg:                                120.10ms
         max:                                936.55ms
         approx.  95 percentile:             252.41ms

Threads fairness:
    events (avg/stddev):           1665.2200/55.49
    execution time (avg/stddev):   199.9870/0.04
    
    
    
    sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql run
    
OLTP test statistics:
    queries performed:
        read:                            1440810
        write:                           514498
        other:                           205798
        total:                           2161106
    transactions:                        102883 (514.03 per sec.)
    deadlocks:                           32     (0.16 per sec.)
    read/write requests:                 1955308 (9769.31 per sec.)
    other operations:                    205798 (1028.23 per sec.)

General statistics:
    total time:                          200.1480s
    total number of events:              102883
    total time taken by event execution: 20005.2891
    response time:
         min:                                 28.62ms
         avg:                                194.45ms
         max:                               1756.56ms
         approx.  95 percentile:             401.20ms

Threads fairness:
    events (avg/stddev):           1028.8300/92.85
    execution time (avg/stddev):   200.0529/0.05




 mysql:
 
sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=5 --max-requests=100000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test run


sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=5 --max-requests=100000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test prepare


OLTP test statistics:
    queries performed:
        read:                            1400000
        write:                           500000
        other:                           200000
        total:                           2100000
    transactions:                        100000 (850.09 per sec.)
    deadlocks:                           0      (0.00 per sec.)
    read/write requests:                 1900000 (16151.68 per sec.)
    other operations:                    200000 (1700.18 per sec.)

General statistics:
    total time:                          117.6348s
    total number of events:              100000
    total time taken by event execution: 587.1712
    response time:
         min:                                  2.32ms
         avg:                                  5.87ms
         max:                                324.33ms
         approx.  95 percentile:              13.09ms

Threads fairness:
    events (avg/stddev):           20000.0000/355.65
    execution time (avg/stddev):   117.4342/0.01



sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=50 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test prepare


OLTP test statistics:
    queries performed:
        read:                            2529044
        write:                           903182
        other:                           361272
        total:                           3793498
    transactions:                        180626 (903.02 per sec.)
    deadlocks:                           20     (0.10 per sec.)
    read/write requests:                 3432226 (17158.97 per sec.)
    other operations:                    361272 (1806.13 per sec.)

General statistics:
    total time:                          200.0252s
    total number of events:              180626
    total time taken by event execution: 9998.0727
    response time:
         min:                                  2.38ms
         avg:                                 55.35ms
         max:                                626.14ms
         approx.  95 percentile:             137.18ms

Threads fairness:
    events (avg/stddev):           3612.5200/45.09
    execution time (avg/stddev):   199.9615/0.02




sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test run


GRANT ALL PRIVILEGES ON *.* TO 'root'@'paxos%' IDENTIFIED BY '' WITH
      GRANT OPTION;  
FLUSH   PRIVILEGES; 
  

OLTP test statistics:
    queries performed:
        read:                            2456258
        write:                           877080
        other:                           350833
        total:                           3684171
    transactions:                        175386 (876.70 per sec.)
    deadlocks:                           61     (0.30 per sec.)
    read/write requests:                 3333338 (16662.36 per sec.)
    other operations:                    350833 (1753.71 per sec.)

General statistics:
    total time:                          200.0520s
    total number of events:              175386
    total time taken by event execution: 19993.1738
    response time:
         min:                                  2.36ms
         avg:                                114.00ms
         max:                               1398.17ms
         approx.  95 percentile:             287.00ms

Threads fairness:
    events (avg/stddev):           1753.8600/35.64
    execution time (avg/stddev):   199.9317/0.06


#remote
./sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test run

OLTP test statistics:
    queries performed:
        read:                            2002532
        write:                           715099
        other:                           286041
        total:                           3003672
    transactions:                        143003 (714.53 per sec.)
    deadlocks:                           35     (0.17 per sec.)
    read/write requests:                 2717631 (13578.85 per sec.)
    other operations:                    286041 (1429.23 per sec.)

General statistics:
    total time:                          200.1371s
    total number of events:              143003
    total time taken by event execution: 20004.6611
    response time:
         min:                                  5.82ms
         avg:                                139.89ms
         max:                               1130.38ms
         approx.  95 percentile:             362.70ms

Threads fairness:
    events (avg/stddev):           1430.0300/32.37
    execution time (avg/stddev):   200.0466/0.07


写3500
读10000




/home/zhangwusheng/phxsql/percona.src/bin/mysql  -uroot -h 10.199.201.143  -P54321


GRANT ALL PRIVILEGES ON *.* TO 'root'@'paxos%' IDENTIFIED BY '' WITH
      GRANT OPTION;  
      
      GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.199.202.222' IDENTIFIED BY '' WITH
      GRANT OPTION; 
      
FLUSH   PRIVILEGES; 




./sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=54321 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql run

OLTP test statistics:
    queries performed:
        read:                            1539664
        write:                           549803
        other:                           219922
        total:                           2309389
    transactions:                        109946 (549.44 per sec.)
    deadlocks:                           30     (0.15 per sec.)
    read/write requests:                 2089467 (10441.79 per sec.)
    other operations:                    219922 (1099.03 per sec.)

General statistics:
    total time:                          200.1062s
    total number of events:              109946
    total time taken by event execution: 20003.1961
    response time:
         min:                                 36.86ms
         avg:                                181.94ms
         max:                               1063.39ms
         approx.  95 percentile:             363.24ms

Threads fairness:
    events (avg/stddev):           1099.4600/186.48
    execution time (avg/stddev):   200.0320/0.06





./sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=11111 --mysql-user=root --mysql-password="" --mysql-db=test_phxsql run

OLTP test statistics:
    queries performed:
        read:                            1539664
        write:                           549803
        other:                           219922
        total:                           2309389
    transactions:                        109946 (549.44 per sec.)
    deadlocks:                           30     (0.15 per sec.)
    read/write requests:                 2089467 (10441.79 per sec.)
    other operations:                    219922 (1099.03 per sec.)

General statistics:
    total time:                          200.1062s
    total number of events:              109946
    total time taken by event execution: 20003.1961
    response time:
         min:                                 36.86ms
         avg:                                181.94ms
         max:                               1063.39ms
         approx.  95 percentile:             363.24ms

Threads fairness:
    events (avg/stddev):           1099.4600/186.48
    execution time (avg/stddev):   200.0320/0.06
    
    
    OLTP test statistics:
    queries performed:
        read:                            1589812
        write:                           567695
        other:                           227077
        total:                           2384584
    transactions:                        113519 (567.16 per sec.)
    deadlocks:                           39     (0.19 per sec.)
    read/write requests:                 2157507 (10779.29 per sec.)
    other operations:                    227077 (1134.52 per sec.)

General statistics:
    total time:                          200.1529s
    total number of events:              113519
    total time taken by event execution: 20009.2781
    response time:
         min:                                 21.92ms
         avg:                                176.26ms
         max:                               1247.89ms
         approx.  95 percentile:             345.01ms

Threads fairness:
    events (avg/stddev):           1135.1900/13.74
    execution time (avg/stddev):   200.0928/0.06
    
    
    222上测试，phxsql部署（143，144，145），percona部署在143：
    直连 percona  写：3575，读 10012
    percona配置了半同步：写 2667 读 7470
    连phxsql proy：写 2749 读 7698
    绕过proxy连 binlogsvr ：写 2838 读：7698
    
   
    
    
    
    bentongbu:
    
    ./sysbench --test=oltp --oltp-num-tables=10 --oltp-table-size=10000 --num-threads=100 --max-requests=500000 --report-interval=1 --max-time=200 --mysql-host=10.199.201.143 --mysql-port=4444 --mysql-user=root --mysql-password="" --mysql-db=test run
    
     queries performed:
        read:                            1494178
        write:                           533593
        other:                           213437
        total:                           2241208
    transactions:                        106710 (533.50 per sec.)
    deadlocks:                           17     (0.08 per sec.)
    read/write requests:                 2027771 (10137.89 per sec.)
    other operations:                    213437 (1067.08 per sec.)

General statistics:
    total time:                          200.0191s
    total number of events:              106710
    total time taken by event execution: 19999.9486
    response time:
         min:                                  5.20ms
         avg:                                187.42ms
         max:                               1401.89ms
         approx.  95 percentile:             588.69ms

Threads fairness:
    events (avg/stddev):           1067.1000/19.76
    execution time (avg/stddev):   199.9995/0.01
