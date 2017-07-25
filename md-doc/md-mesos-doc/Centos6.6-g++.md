Centos安装g++4.9


cd /etc/yum.repos.d
wget http://linuxsoft.cern.ch/cern/scl/slc6-scl.repo
yum -y --nogpgcheck install devtoolset-3-gcc devtoolset-3-gcc-c++


