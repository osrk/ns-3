# ns-3

Windows
g++をインストールした
CMake.txtを変更した

tar jxf ../ns-allinone-3.39.tar.bz2
cd ns-allinone-3.39/ns-3.39/
./ns3 configure --enable-examples --enable-tests

error

```
sudo apt-get update
sudo apt-get install ninja-build
sudo apt-get install make
sudo apt-get install cmake
sudo apt-get install gcc
sudo apt install g++

./ns3 configure --enable-examples --enable-tests
```

```
vi CMakeLists.txt


```
cmake_minimum_required(VERSION 3.10..3.10)

set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
```

```
./ns3 configure --enable-examples --enable-tests
./ns3 build
```
