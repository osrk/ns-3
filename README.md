# ns-3

## Windows WSL インストール

```
tar jxf ../ns-allinone-3.39.tar.bz2
cd ns-allinone-3.39/ns-3.39/
./ns3 configure --enable-examples --enable-tests
```

エラー発生したので、必要そうなパッケージをインストールする。

```
sudo apt-get update
sudo apt-get install ninja-build
sudo apt-get install make
sudo apt-get install cmake
sudo apt-get install gcc
sudo apt install g++

./ns3 configure --enable-examples --enable-tests
```

またエラー発生

```
-- Configuring done
CMake Error at build-support/custom-modules/ns3-module-macros.cmake:81 (add_library):
  The install of the libantenna target requires changing an RPATH from the
  build tree, but this is not supported with the Ninja generator unless on an
  ELF-based or XCOFF-based platform.  The CMAKE_BUILD_WITH_INSTALL_RPATH
  variable may be set to avoid this relinking step.
Call Stack (most recent call first):
  src/antenna/CMakeLists.txt:1 (build_lib)
```

メッセージ `The CMAKE_BUILD_WITH_INSTALL_RPATH variable may be set to avoid this relinking step.` のとおりに設定する。

```
vi CMakeLists.txt
```

```
cmake_minimum_required(VERSION 3.10..3.10)

set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
```

```
./ns3 configure --enable-examples --enable-tests
./ns3 build
./test.py 
```
