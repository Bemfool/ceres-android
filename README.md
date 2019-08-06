# ceres-android

在Android平台上使用Ceres求解器，官方教程不明确，且编译过程遇到了很多问题。

## 环境

Ubuntu 18.04

##　准备工作

[Eigen 3.3.7](http://eigen.tuxfamily.org/index.php?title=Main_Page)（最新）：在编译Ceres的时候需要使用。

[NDK r20](https://developer.android.google.cn/ndk/downloads/)（最新）：NDK r14b版本无法使用，建议使用高于14的版本。

[Ceres 1.14.0](http://ceres-solver.org/installation.html)（最新）已经附带在仓库中。

*[NOTE] Ceres其他依赖项按照官方教程进行配置，建议先检验Linux系统下是否能够安装成功。* 

```shell
# 官方安装指导 http://ceres-solver.org/installation.html

# CMake
sudo apt-get install cmake
# google-glog + gflags
sudo apt-get install libgoogle-glog-dev
# BLAS & LAPACK
sudo apt-get install libatlas-base-dev
# Eigen3
sudo apt-get install libeigen3-dev
# SuiteSparse and CXSparse (optional)
# - If you want to build Ceres as a *static* library (the default)
#   you can use the SuiteSparse package in the main Ubuntu package
#   repository:
sudo apt-get install libsuitesparse-dev
# - However, if you want to build Ceres as a *shared* library, you must
#   add the following PPA:
sudo add-apt-repository ppa:bzindovic/suitesparse-bugfix-1319687
sudo apt-get update
sudo apt-get install libsuitesparse-dev

mkdir ceres-bin
cd ceres-bin
cmake ../ceres-solver-1.14.0
make -j3
make test
# Optionally install Ceres, it can also be exported using CMake which
# allows Ceres to be used without requiring installation, see the documentation
# for the EXPORT_BUILD_DIR option for more information.
make install
```

Android Studio 3.4.1：需要配置`CMake`, `LLDB`, `ndk`（在`SDK Tools`中点击安装）



## 官方Android指南

### [来源一：官方网站](http://ceres-solver.org/installation.html#android)

#### 原文

下载版本高于`r9d`的`Android NDK`版本，在`jni`目录下使用`ndk-build`进行编译，你会得到`libceres.a`。

#### 解释

具体使用方法：

``` shell
cd ceres-solver-1.14.0/jni
EIGEN_PATH=/path/to/eigen/header ndk-build
```

随后`libceres.a`便会出现在`ceres-solver-1.14.0/obj/local/${ABI}`目录下。

`${ABI}`的值通过修改`Application.mk`中的`APP_ABI := arm64-v8a`一行，可.以换`armeabi-7va`, `x86_64`等等。

至于动态库（*.so）的生成，根据网上的一些教程进行修改`Android.mk`和`Application.mk`，都无法正常编译，官方GitHub里的issue中对于这些报错，建议使用CMake进行编译。

*[NOTE] 如果从GitHub上clone的版本是不带jni目录的，只能使用CMake进行编译。* 

### [来源二：代码内的文档](https://github.com/ceres-solver/ceres-solver/blob/master/docs/source/installation.rst#android)

#### 原文

编译需要使用NDK r15或者更高版本。

需要使用CMake寻找NDK当中的toolchain来替换本身的自带toolchain。假设你已经设置了变量`$NDK_DIR`，你可以使用如下命令编译：

```shell
cmake \
-DCMAKE_TOOLCHAIN_FILE=\
    $NDK_DIR/build/cmake/android.toolchain.cmake \
-DEIGEN_INCLUDE_DIR=/path/to/eigen/header \
-DANDROID_ABI=armeabi-v7a \
-DANDROID_STL=c++_shared \
-DANDROID_NATIVE_API_LEVEL=android-24 \
-DBUILD_SHARED_LIBS=ON \
-DMINIGLOG=ON \
<PATH_TO_CERES_SOURCE>
```

你可以为各种安卓STL或ABI进行编译，但是最推荐使用c++_shared STL和armeabi-v7a（对于32位机）或arm64-v8a（对于64位机） ABI。许多API的版本都可以进行支持，但是推荐使用你安卓项目所支持的最高版本。

对于你的安卓项目和Ceres二进制文件，你需要使用相同的API版本和STL进行编译。

编译完成之后，你会得到一份`libceres.so`的库，你可以在编译的脚本中通过使用`PREBUILT_SHARED_LIBRARY`将它链接到你的安卓项目当中。

如果你正在编译Ceres的例子，想要验证你的库能否使，你需要将他们和`libceres.so`放在安卓设备上一个可执行的公共目录下（比如`/data/local/tmp`），并且确保NDK的STL也在同一个目录下。

需要注意的是，所有的求解器或者其他被你包含在项目中的共享依赖项，都需要出现在你安卓的编译配置和你的测试目录当中。

#### 解释

最好采用CMake编译这种方法，后面解释如何修改这个命令。

## 编译命令

参考资料：https://github.com/qiu-yongheng/cerestest

根据官方材料和参考资料得出如下命令：

```shell
rm -rf CMake*

/path/to/sdk/cmake/[version]/bin/cmake \
-DCMAKE_TOOLCHAIN_FILE=/path/to/sdk/ndk-bundle/build/cmake/android.toolchain.cmake \
-DEIGEN_INCLUDE_DIR=/path/to/eigen/header \
-DANDROID_ABI=[ABI]] \
-DANDROID_STL=[c++_shared | c++_static] \
-DANDROID_NATIVE_API_LEVEL=android-24 \
-DBUILD_SHARED_LIBS=ON \
-DMINIGLOG=ON \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_C_FLAGS="-s" \
-DCMAKE_C_FLAGS=-std=c99 -Os -fvisibility=hidden \
/path/to/ceres

make clean
make
make install
```

执行如上命令后，首先会遇到如下错误：`undefined reference to '__android_log_write'`

解决方法：打开`[ceres]/internal/ceres/minglog/glog/logging.h`，搜索`__android_log_write`，将带有这个函数的几行注释掉。

重新编译，最后会报错误：`'libc++_shared.so' no found`

解决方法：这个错误无关紧要，可以看到当前目录下的`lib`目录已经生成了我们想要的库文件。从`ndk-bundle`中复制我们所需要的该文件到安卓项目中即可，该文件目录可能是`ndk-bundle/sources/cxx-stl/llvm-libc++/libs/[ABI]`。无需继续进行编译。

*[NOTE] `/path/to/eigen/header`这个地址，不能使用安装`eigen`时候放置到的比如`/usr/include`地址，否则编译的时候会出现冲突，应该直接使用下载解压后的位置。*

*[NOTE] 每次cmake后，重新编译需要提前先删除当前目录下的`CMakeFiles`,  ` CMakeCache`等中间文件，否则CMake的配置不会发生改变。*



## 使用过程

修改（新建）`build`目录下`config.sh`脚本，内容如上CMake命令，将地址修改为对应的目录。

```shell
./config.sh
```

运行后得到`libceres.so`或者`libceres.a`，将其放到安卓项目目录下（如`android-studio-example/cerestest/app/src/main/jniLibs/[ABI]`）。

然后就可以通过修改对应的`cpp`文件来编写`JNI`函数。



## 测试结果

![](https://img2018.cnblogs.com/blog/1368985/201908/1368985-20190806101249587-368900254.jpg)