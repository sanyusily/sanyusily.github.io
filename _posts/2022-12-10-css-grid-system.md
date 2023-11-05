---
layout: article
title: '常用命令 Linux npm'
tags: Linux
---


# 实际开发中常用的




# 实际开发中常用的Linux命令和Android 指令

  
~~~css
- vim ~/.bashrc
~~~

## NDK添加如下信息

~~~NDK
export NDKROOT=/ndk解压目录/android-ndk-r21
export PATH=$NDKROOT:$PATH
~~~

  
## CMAKETEXT添加如下信息
~~~NDK
cmake -S . -B build

cmake --build build



cmake -S . -B build -DBUILD_SHARED_LIBS=YES

cmake --build build

app.thread.scheduleLaunchActivity


add_executable()

add_library() 命令支持可选的三个互斥参数：STATIC | SHARED | MODULE

cmake -DBUILD_SHARED_LIBS=YES

~~~

  


## 如何在Linux下安装cmake

~~~shell

sudo apt-get install cmake

  
  


如果没有安装g++,gcc,gdb，输入以下命令进行安装

sudo apt-get install build-essential

sudo apt install gdb
~~~
  
  


## 制作android系统线程方法

- 放在根目录，然后make xxx目标（去mk中看

~~~shell

adb push out/target/product/generic_x86/system/bin /data/local/

./android_thread



  


man abort

stdlib.h

~~~
  


the IDE is running low on memory

## 日志打印技巧：

~~~
adb shell

logcat -b all | grep hello

Log.i(TAG,"message",new Exception())

全局搜索字符串 grep "\.handleMeassage" ./ -rn

adb shell dumpsys activity activities

  
~~~



# 升级 npm


~~~
sudo apt update

sudo apt install nodejs

#不自带 npm 需要自行安装

sudo apt install npm


sudo npm install npm -g
  
  
//install compare software

https://www.scootersoftware.com/support.php?zz=kb_linux_install


buildToolsVersion = "33.0.1"

compileSdkVersion 33

sudo npm install n -g

# 下载最新稳定版

sudo n stable

# 下载最新版

sudo n lastest

# 查看已下载的版本

sudo n ls

# 切换 Node 版本

sudo n 18.21.1

~~~



# 安装React Native命令行界面。

|| appProject?.ext?.react?.enableHermes?.toBoolean()


~~~

npm install --legacy-peer-deps 解决安装问题

npm install -g react-native-cli

npm uninstall -g react-native-cli @react-native-community/cli

  
  


cd AwesomeProject

react-native run-ios

  


npm install redux // 生产环境的依赖

npm install redux -dev // 开发环境的依赖

npm uninstall redux
~~~

~~~
$ cd android

$ ./gradlew assembleRelease

// 实例化 metro Server启动 metro 构建 bundle处理资源文件

commandLine(*execCommand, bundleCommand, "--platform", "android", "--dev", "${devEnabled}",

"--reset-cache", "--entry-file", entryFile, "--bundle-output", jsBundleFile, "--assets-dest", resourcesDir,

"--sourcemap-output", enableHermes ? jsPackagerSourceMapFile : jsOutputSourceMapFile, *extraArgs)

ios打包：

npx react-native run-ios --configuration Release

shellScript = "export NODE_BINARY=node\n#export FORCE_BUNDLING=true\nexport BUNDLE_COMMAND=bundle\n../node_modules/
react-native/scripts/react-native-xcode.sh\n";

  
  


https://gitee.com/nuogui/AndrCodeRepository.git

  


ps -ef | grep wechat | cut -c 11-16 | xargs sudo kill -9

  


ps -ef | grep -v wechat | cut -c 12-18  
~~~


# 安装闹钟

- Find Pomodoro in Software Center or run


~~~
#git clone -b gnome-41 https://github.com/gnome-pomodoro/gnome-pomodoro.git

sudo apt-get install gnome-shell-pomodo

安装命令 sudo dpkg -i 文件名.deb

启动命令 /usr/local/sunlogin/bin/sunloginclient

卸载命令 sudo dpkg -r sunloginclient

  ~~~
  
  
~~~
uname -v

lsb_release -a


http://us.archive.ubuntu.com/ubuntu xenial InRelease: 由于没有公钥，无法验证下列签名： NO_PUBKEY 40976EAF437D05B5 
NO_PUBKEY 3B4FE6ACC0B21F32



sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 40976EAF437D05B5

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32

  
~~~
  

~~~

1.查看安装的所有软件

dpkg -l

例如：dpkg -l | grep ftp

  


2.查看软件安装的路径

dpkg -L | grep ftp

也可以用 whereis ftp

  


3.查看软件版本

aptitude show xxx

  


安装应用时使用

apt-get install

卸载应用时使用

sudo apt remove program_name

完全卸载应用

sudo apt purge program_name

如果不知道具体的应用名称或者开头字母，你可以 列出 Ubuntu 中所有已安装的包

apt list --installed | grep -i xxx

  


snap list

  


nano

字符终端文本编辑器

sudo nano /etc/pam.d/common-password

使用Ctrl+O来保存所做的修改 退出 按Ctrl+X

  


sudo passwd root 给root用户修改密码

su root 切换用户

  


root密码：52101985zzMzzM*

  
~~~


repo安装

~~~shell

sudo apt install repo

  


git config --global user.email "teitibou@163.com"

git config --global user.name "teitibou"

  
  


sudo gedit /etc/apt/sources.list //在行尾添加如下两行的内容

deb http://us.archive.ubuntu.com/ubuntu/ xenial main universe

deb-src http://us.archive.ubuntu.com/ubuntu/ xenial main universe

  


cat /proc/cpuinfo来查看相关的cpu信息

lsb_release -a，查看发行版本信息，并且方法可以适用于所有的Linux发行版本。

  


cat /etc/issue可以查看到当前是Linux什么版本系统。

cat /proc/version可以查看内核的版本号。

  


which java 查找java

ls -lrt /usr/bin/java 查找文件快捷链接原地址

ln -s /usr/local/jvm/jdk1.8.0_303/ /usr/jdk 创建一个软连接

  


/usr/lib/jvm/java-11-openjdk-amd64/bin/java

  


du -sh ./folder ubuntu 查看文件夹大小

/usr/lib/jvm/java-11-openjdk-amd64

  


// 打开/etc/apt/sources.list

sudo nano /etc/apt/sources.list

//在最后粘贴以下两行

deb http://us.archive.ubuntu.com/ubuntu/ xenial main universe

deb-src http://us.archive.ubuntu.com/ubuntu/ xenial main universe

//ctrl +s 保存，ctrl + x退出

//在terminal执行

sudo apt-get update && sudo apt-get install libesd0-dev

  
  


工作有意义，生活有品质 git 操作operator

sanyusily.github.io/mdbook_site

  


$ git remote add origin https://github.com/sanyusily/sanyusily.github.io.git

$ git remote add origin git@github.com:sanyusily/sanyusily.github.io.git

  


$ git stash

$ git pull --rebase origin master

# change to your github repo

$ git stash pop

$ git rebase --continue

  


$ git branch -M develop

$ git push -u origin develop

  


mkdir quickstart-react-native

cd quickstart-react-native

git init

touch README.md

git add README.md

git commit -m "first commit"

git remote add origin https://gitee.com/sanyusily/quickstart-react-native.git

git push -u origin "master"

  
  
  


git branch --set-upstream-to=origin/master master

因为远程仓库新建时，有LIENCE，由于本地仓库和远程仓库有不同的开始点，也就是两个仓库没有共同的commit出现，无法提交，此时我们
allow-unrelated-histories。也就是我们的 pull 命令改为下面这样的：

git pull origin master --allow-unrelated-histories

  


git remote -v

git remote add 仓库地址

git remote rm 仓库名称

  


$ ssh-keygen -t rsa

$ cat ~/.ssh/id_rsa.pub

  


$ code . 打开vscode

  


解决git每次提交代码都要输入帐号密码

git config --global credential.helper store

  
  


github.com/oh-my-docker/jekyll-docker

  


$ sudo apt install docker.io

  
  
  


$ sudo apt-get install ruby ruby-dev 安装Ruby包和开发工具：

$ sudo apt-get install make build-essential 必需的构建工具

$ sudo gem install bundler 依赖包安装

$ sudo gem install jekyll

bundle exec jekyll -v

bundle exec jekyll serve

  


卸载重新安装jekyll

PACKAGES="$(dpkg -l |grep jekyll|cut -d" " -f3|xargs )"

sudo apt remove --purge $PACKAGES

sudo apt autoremove

sudo gem install jekyll jekyll-feed jekyll-gist jekyll-paginate jekyll-sass-converter jekyll-coffeescript

bundle update

  


bundle exec jekyll new myblog

bundle exec jekyll serve

  
  
  
  


https://github.com/oh-my-docker/jekyll-docker

  


docker pull quay.io/jetstack/cert-manager-cainjector:v0.12.0

docker pull quay-mirror.qiniu.com/jetstack/cert-manager-cainjector:v0.12.0

  


docker pull quay-mirror.qiniu.com/omd/jekyll

$ docker pull quay.io/omd/jekyll

$ docker pull quay-mirror.qiniu.com/omd/jekyll



2023年最新最全 VSCode 插件推荐！
VSCode React Refactor
Simple React Snippets
Type React Code Snippets
npm Intellisense  该插件为 import 语句中的 npm 模块提供了自动完成功能。npm 模块的所有导入都会使用此扩展自动处理。
Path intellisense 该插件用于自动补全文件名。当 import 其它文件时，能够对文件进行提示，快速补全要引入的文件名。
Better comments 该插件对不同类型的注释会附加了不同的颜色

Git Graph  插件用于可视化查看存储库的 Git 操作，并从图形中轻松执行Git操作

## 使用 Jekyll 获取源代码 使用主题让博客更美观
~~~

友情引用

~~~css
https://www.sohu.com/a/644680998_121124376

https://www.youtube.com/channel/UCmlhPmTdqYhRWwWZWSIBwGw/featured

https://gist.github.com/biezhi

http://open-developer.blogspot.com/search?q=jekyll

https://github.com/myanbin/myanbin.github.io
~~~

  