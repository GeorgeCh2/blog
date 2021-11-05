# [自己动手编译一个HEIF图片转jpeg工具(Mac平台)](https://github.com/GeorgeCh2/blog/issues/6)

随着 Android 10 对HEIF图片格式的支持，后续将有越来越多的移动设备支持HEIF图片的拍摄。但是部分Windows设备，老的移动设备不支持HEIF图片的查看，下面介绍如何在Mac上自己编译一个HEIF转jpeg的可执行程序。

### 环境搭建

首先mac需要安装 `automake`、`make`、`libtool`，可通过homebrew进行安装

```
brew install automake,make,libtool
```



### 下载开源库

1. libjpeg-turbo: https://github.com/libjpeg-turbo/libjpeg-turbo
2. libde265:https://github.com/strukturag/libde265
3. libheif: https://github.com/strukturag/libheif



### 对源码进行编译

#### 编译 jpeg（用于jpeg图片编码）

在jpeg源码目录下

```shell
rm -rf CMakeCache.txt
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DENABLE_SHARED=OFF -DBUILD_SHARED_LIBS=OFF -DENABLE_CLI=OFF 

make clean
make -j4
make install
```



#### 编译 de265（用于HEIF图片解码）

在 de265 源码目录下

```shell
./autogen.sh
./configure --prefix=${INSTALL_DIR} \
            --enable-static \
            --disable-shared \
            --disable-encoder \
            --enable-log-error \
            --disable-dec265 \
            --disable-sherlock265

make clean
make -j4
make install
```



#### 编译 libheif

在 libheif 源码目录下

```shell
./autogen.sh
export PKG_CONFIG_PATH="${INSTALL_ROOT}/de265/lib/pkgconfig"

./configure CPPFLAGS="-I${INSTALL_ROOT}/jpeg/include" \
            LDFLAGS="-L${INSTALL_ROOT}/jpeg/lib64" \
            --prefix=${INSTALL_DIR} \
            --enable-static \
            --disable-shared 
        
make clean
make -j4
make install 
```



### 转换 HEIF

libheif 会编译出 heif-convert、heif-info、heif-enc 三个可执行文件

heif-convert : 转换HEIF文件（由于只将jpeg库编译进来，只支持转换成jpeg）

heif-info：查看hief文件信息

heif-enc：支持转换成 heif文件（因为没有编译h265编码库，目前不支持生成heif）

如果想要转换成jpeg，可以直接使用 heif-convert (quality 为质量参数，越大质量越好。filename 为heif文件路径，output 为输出jpeg文件路径)

```
./heif-convert [-q quality 0..100] <filename> <output>
```



> 上面编译的脚步已经整理到了github：https://github.com/GeorgeCh2/libheif-static-complie