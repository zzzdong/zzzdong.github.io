+++
date = '2025-04-14T10:41:51+08:00'
draft = false
title = '搭建Rust的Windows GNU开发环境'
+++


## 前言

在Windows上，使用Rust开发时，官方是推荐MSVC的环境，但是MSVC环境需要VS，或者至少Microsoft C++ 生成工具。这里需要注意的一点，就是它们的许可问题，在我们常用的Community版本中，是有比较多的限制的，它要求在开源项目，或者是非企业环境中。

这时，gnu版本是一个不错的选择，它相对没有那么多的约束。

只不过它的安装会有些麻烦。

## 安装gnu环境

1. 通过rustup安装rustc。

下载 https://static.rust-lang.org/rustup/dist/x86_64-pc-windows-gnu/rustup-init.exe ，双击安装。使用直接回车默认配置安装即可。

当已有的rustup的时候，可以：

```
rustup toolchain install stable-x86_64-pc-windows-gnu
rustup default stable-x86_64-pc-windows-gnu
```

2. 安装mingw-w64开发工具。

根据 https://www.mingw-w64.org/downloads/#w64devkit 的说明，下载安装w64devkit开发工具。

下载后，运行exe，选择一个目录解压安装。

配置环境变量，把上面的安装目录的bin目录添加到PATH中。

3. 修正w64devkit的兼容性问题。

上面安装后，运行cargo build可能会报错，出现`ld.exe: cannot find -lgcc_eh`相似的错误。这时，需要手动修正一下。

生成一个空的`libgcc_eh.a`文件。

```shell
cd w64devkit/x86_64-w64-mingw32/lib
ar rcs libgcc_eh.a
```


## 参考资料

1. https://github.com/stacks-network/stacks-core/pull/5937#issuecomment-2747629024
