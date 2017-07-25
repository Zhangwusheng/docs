#Centos6.6

cd /etc/yum.repos.d
wget http://linuxsoft.cern.ch/cern/scl/slc6-scl.repo
yum -y --nogpgcheck install devtoolset-3-gcc devtoolset-3-gcc-c++
scl enable devtoolset-3 bash

source /opt/rh/devtoolset-3/enable

#CentOS 6.7编译mesos


wget http://apache.fayea.com/mesos/1.0.1/mesos-1.0.1.tar.gz

cat /etc/centos-release 
--CentOS release 6.7 (Final)


yum install centos-release-scl


yum install devtoolset-3-toolchain


wget http://apache.fayea.com/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz


tar zxvf apache-maven-3.3.9-bin.tar.gz  -C /usr/local
ln -fs /usr/local/apache-maven-3.3.9 /usr/local/maven


vi /etc/profile.d/maven.sh

export MAVEN_HOME=/usr/local/maven
export PATH=$PATH:${MAVEN_HOME}/bin


将
source /opt/rh/devtoolset-3/enable
添加到 .bash_profile最后面



yum install python-devel


yum -y install pcre-devel

yum install zlib-static

yum install zlib-devel

yum install cyrus-sasl-md5

yum install  apr-util-devel

yum install  apr-devel

yum install  libcurl-devel

yum install subversion-devel

yum install subversion



make -j 8



