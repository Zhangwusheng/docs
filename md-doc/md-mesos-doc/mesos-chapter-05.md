

##使用XCode调试Mesos
http://blog.burhum.com/post/35555678746/debugging-makefile-based-projects-using-xcode

Debugging Makefile-based projects using XCode (GDAL as example)

I love using XCode to write C++. Nevertheless, many of the projects I use a lot (like GDAL) don’t maintain XCode projects. Compiling these type of projects usually is usually done with autoconf and GNUMakefiles. The usual workflow takes this form from the command line:

  ./configure
  ./make
  ./make install
But what if I want to use XCode? Here is a quick way I use to get up and running with XCode 4.5.2 Let’s use GDAL as an example.

Create a new XCode project

1.- File->New Project

2.- Choose Application -> Command Line Tool. Call it something meaningful like gdal-trunk (uncheck use automatic reference counting).

3.- Delete the Target

Checkout the version you want from GDAL and compile it with debug

First, we need to make sure that our project is compiled with the -g flag so debugging information is included. For GDAL, this is accomplished with the –enable-debug flag. Personally, I also like to keep my debug build separate from the rest of my system. So I also specify a build directory using the –prefix flag.

These are the commands I use in the command line:

 svn checkout https://svn.osgeo.org/gdal/trunk/gdal gdal
 cd gdal
 ./configure --enable-debug --prefix=/Users/rburhum/src/gdal-xcode/gdal-trunk/build
 make
 make install
Setup XCode to show GDAL source and call the makefile every time we build

1.- While highlighting the project: File->New Taget. Under OSX -> Other, choose “External Build System”. Pick a target name like “gdal_target”. Finish.

2.- Click on the new target, select info. Make sure the directory there has the root directory that has your makefile. In my case /Users/rburhum/src/gdal-xcode/gdal

3.- Product->Edit Scheme. Under Build->Build hit the “+” and add the new target you just created. This will make sure that make gets called. But with GDAL, you will need to have the code installed again after every compilation (a different topic for another day). So we will need to trigger that behavior:

4.- Under Post Actions, add the following script: cd /Users/rburhum/src/gdal-xcode/gdal /usr/bin/make install

Setup XCode to call the executable you want to debug

Almost there! The last step missing is to add the executable you want to debug. Let’s say that it is ogrinfo.

While still on the Edit Scheme window. Choose Run. Under Info, browse and pick ogrinfo from your build directory. Under Arguments, choose whatever you may want to pass to ogrinfo (in my case:

  -al /Users/rburhum/Applications/geoserver-2.1.3/data_dir/data/nyc/poi.shp
Add the GDAL source tree to XCode

Right click on your Project and Choose “Add files to gdal-trunk”. Make sure you choose “Create Folder References” and that “Copy items into destination folder” is unchecked. Browse to the root of the GDAL directory and pick it.

You are done! Put your breakpoints, change your code and whenever you hit run/debug, XCode will compile the new code and run it inside XCode.

***注：最后编译出来的一般都放到 .libs下面，隐藏目录在mac下面不好操作，最好在编译的POST那里把这些文件mv到一个可见目录，比如libs下面，然后调试的时候直接使用这些程序调试。***




***这时候代码不能jump to definition等，可以新建一个target
File->New->Target
然后，在每个文件夹上，Sort By Type，把所有的cpp都加入到这个target里面去，然后就可以跳转了。
摸索了好几天，搜索了好几天：）




