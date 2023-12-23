---
layout: default
title: MLIR环境配置
nav_order: 2
---

# Ubuntu22.04 MLIR开发环境文档

## 更新记录

时间|版本|人员|记录
-|-|-|-
2023/12/13|V1|时辰|wsl ubuntu 22.04环境下首次安装MLIR记录整理成文档

## 依赖环境配置

更换ubuntu系统apt源为重大镜像地址：

```bash
sudo sed -i 's/archive.ubuntu.com/mirrors.cqu.edu.cn/g' /etc/apt/sources.list
```

更新源：

```bash
sudo apt update
```

软件依赖环境：

Package|Version|Notes
-|-|-
CMake|>=3.20.0|Makefile/workspace generator
python|>=3.6|Automated test suite1
zlib|>=1.2.3.4|Compression library2
GNU Make|3.79, 3.79.1|Makefile/build processor3

使用如下命令安装依赖：

```bash
sudo apt install cmake clang ninja-build lld build-essential 
```

默认的 Ubuntu 软件源包含了一个“build-essential”软件包组，包含了 GNU 编辑器集合，GNU 调试器，和其他编译软件所必需的开发库和工具。这个命令将会安装cmake、clang、ninja、lld和一系列软件包，包括gcc,g++,和make。

### wsl存在的问题

*安装lld时存在问题：windows wsl子系统下可能会出现错误信息：/sbin/ldconfig.real: /usr/lib/wsl/lib/libcuda.so.1 is not a symbolic link，此时可输入以下命令解决：*

```bash
sudo cat >>/etc/wsl.conf <<EOF
[automount]
ldconfig=false
EOF
```

> 参考：<https://blog.csdn.net/qq_42756195/article/details/125769622>

## llvm编译/测试

git获取llvm-project源代码：

```bash
git clone https://github.com/llvm/llvm-project.git
```

并创建llvm-project/build目录，并切换到该目录：

```bash
mkdir llvm-project/build && cd llvm-project/build
```

配置ninja：

```bash
cmake -G Ninja ../llvm \
   -DLLVM_ENABLE_PROJECTS=mlir \
   -DLLVM_BUILD_EXAMPLES=ON \
   -DLLVM_TARGETS_TO_BUILD="Native;host;RISCV" \
   -DCMAKE_BUILD_TYPE=Release \
   -DLLVM_ENABLE_ASSERTIONS=ON \
   -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLLVM_ENABLE_LLD=ON 
#    -DLLVM_CCACHE_BUILD=ON
#    Using clang and lld speeds up the build

```

cmake编译（多线程存在内存溢出问题，不使用多线程编译时间会很漫长）：

```bash
cmake --build . --target check-mlir
```
