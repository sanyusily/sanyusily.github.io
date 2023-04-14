---
layout: article
title: 'Ubuntu20.04编译Android10系统源码'
tags: read
---
Ubuntu20.04编译Android10系统源码
# Ubuntu20.04编译Android10系统源码


## 1. 本地解压方式
下载链接地址：https://pan.baidu.com/s/1Jwsrb-zwrQO-HEHo5eo9Jg 提取码:uu1j
百度云下载相关的源码包，进行本地解压,下载我提供的百度云链接 android-8.1.0_r1

### 1.1 sudo apt-get install p7zip
### 1.2 7zr x android-8.1.0_r1.7z

## 2.安装git 
sudo apt-get install git

## 3.安装curl库
sudo apt-get install curl

## 4.安装python
sudo apt-get install python
### 
4.1 sudo ln -s /usr/bin/python3 /usr/bin/python 创建一个链接符号到 python 命令 

sudo apt install python2
sudo ln -s /usr/bin/python2 /usr/bin/python


## 5.sudo apt-get install openjdk-11-jdk


  https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/ 
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest

sudo nano ~/.bashrc命令进入 nano 编辑器修改
# https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/
export PATH=~/bin:$PATH
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'


sudo apt-get install curl
mkdir ~/bin
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo > ~/bin/repo
chmod a+x ~/bin/repo

//android安装工具
sudo apt install m4 libncurses5 python-is-python3

## 6.安装依赖
sudo apt-get install libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-dev g++-multilib
sudo apt-get install -y git flex bison gperf build-essential libncurses5-dev:i386
sudo apt-get install tofrodos python3-markdown libxml2-utils xsltproc zlib1g-dev:i386
sudo apt-get install dpkg-dev libsdl1.2-dev libesd0-dev
sudo apt-get install git-core gnupg flex bison gperf build-essential
sudo apt-get install zip curl zlib1g-dev gcc-multilib g++-multilib
sudo apt-get install libc6-dev-i386
sudo apt-get install lib32ncurses5-dev x11proto-core-dev libx11-dev
sudo apt-get install libgl1-mesa-dev libxml2-utils xsltproc unzip m4
sudo apt-get install lib32z-dev ccache
sudo apt-get install libssl-dev


sudo gedit /etc/apt/sources.list  //在行尾添加如下两行的内容
deb http://us.archive.ubuntu.com/ubuntu/ xenial main universe
deb-src http://us.archive.ubuntu.com/ubuntu/ xenial main universe


## 7.编译 aosp 代码
- 1、 . build/envsetup.sh
- 2、lunch android10选择22 android8选择6
这里我们选择：6 –-- > aosp_x86_64
- 3、make
 
        make clean----> make
        经历大概几个小时等待


## 8.emulator
- emulator
