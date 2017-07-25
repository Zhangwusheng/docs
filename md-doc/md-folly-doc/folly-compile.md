#编译
-----
## 0.设置ldconfig路径

echo '/usr/local/lib' > /etc/ld.so.conf.d/local.cong

yum install libunwind-devel.x86_64

## 1. 编译gflags

修改CMakeList.txt设置CMAKE_CXX_FLAGS增加 -fPIC -DPIC选项

	make install
	ldconfig

## 2. 编译glog

修改configure脚本，增加 -L/usr/local/lib

	ldconfig
	autoreconf -ivf
	./configure
	make
	make install
## 3.编译boost
此处下载的是boost_1_63_0
./bootstrap.sh
./b2
./b2 install

## 4.编译
git clone git://github.com/bos/double-conversion.git
mkdir double-conversion/double-conversion/objs
cd double-conversion/double-conversion/objs
修改CMakeList.txt
增加
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC -DPIC")
cmake ..
make 
make install

## 5.编译libevent
github下载libevent-release-2.1.8-stable.zip
unzip libevent-release-2.1.8-stable.zip
cd libevent-release-2.1.8-stable
./autogen.sh
./configure
make
make install
## 6.编译folly
	autoreconf -ivf
	./configure
	make
	make install
