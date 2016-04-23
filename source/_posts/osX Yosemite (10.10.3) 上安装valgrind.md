title: osX Yosemite (10.10.3) 上安装valgrind
categories: tool
tags: [tool]
---
### brew 安装出错
```
brew install valgrind
valgrind: OS X Mavericks or older is required.
Error: An unsatisfied requirement failed this build.
```

### 从源码安装
```
svn co svn://svn.valgrind.org/valgrind/trunk valgrind
cd valgrind
./autogen.sh
./configure --prefix=/usr/local
make && make install
```

### 从源码安装需要解决的问题
```
ld: cannot found  -lgcc_s.10.5

# 查找此动态库的路径
sudo find / -iname libgcc_s.10.5.dylib 
# 将找到的路径加入编译时动态库查找路径
export LIBRARY_PATH="/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.10.sdk/usr/lib/"
```
