编译folly

1.wget -c http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.xz


xz -d autoconf-2.69.tar.xz

tar xvf autoconf-2.69.tar


wget http://ftp.gnu.org/gnu/m4/m4-1.4.18.tar.xz


修改 configure.ac:

m4_define([folly_version_str], m4_esyscmd([cat VERSION | tr -d '\n']))

首先安装 gflag，安装到 /usr/local/lib下面
一定要ccmake，选择生成so。


再安装 glog ，编译的时候需要修改一下Makefile，把gflags的链接选项加上  -L /usr/local/lib


安装double-conversion.
double-conversion安装到了/usr/local/lib64，这里需要拷贝到/usr/local/lib下面

安装 gtest，把include的拷贝到 /usr/local/include下面，把库拷贝到/usr/local/lib下面

安装boost-1.63

wget -c https://github.com/libevent/libevent/releases/download/release-2.1.8-stable/libevent-2.1.8-stable.tar.gz

安装libevent 2.1.8-stable
./configure
make
/root/folly/libevent-release-2.1.8-stable/include/event2/event-config.h
注意：cmake编译出来的没有-fPIC不行的

yum install -y libunwind-devel.x86_64

cd folly
autoreconf -ivf
./configure

