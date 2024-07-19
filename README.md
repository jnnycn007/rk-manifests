# 主机构建环境搭建

## Ubuntu LTS
```
# 安装SDK构建所需要的软件包
sudo apt install git ssh make gcc libssl-dev liblz4-tool u-boot-tools curl\
expect g++ patchelf chrpath gawk texinfo chrpath diffstat binfmt-support \
qemu-user-static live-build bison flex fakeroot cmake gcc-multilib g++-multilib \
unzip device-tree-compiler python-pip libncurses5-dev python3-pyelftools \
dpkg-dev

```

## 安装repo

```
mkdir ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
# 如果上面的地址无法访问，可以用下面的：
# curl -sSL  'https://gerrit-googlesource.proxy.ustclug.org/git-repo/+/master/repo?format=TEXT' |base64 -d > ~/bin/repo
chmod a+x ~/bin/repo    
echo PATH=~/bin:$PATH >> ~/.bashrc
source ~/.bashrc
```
执行完上面的命令后我们来验证repo是否安装成功能正常运行。
```
repo --version

#返回以下信息
#返回的信息根据Ubuntu版本的不同略有差异
<repo not installed>
repo launcher version 2.32
    (from /home/he/bin/repo)
git 2.25.1
Python 3.8.10 (default, Nov 14 2022, 12:59:47) 
[GCC 9.4.0]
OS Linux 5.15.0-60-generic (#66~20.04.1-Ubuntu SMP Wed Jan 25 09:41:30 UTC 2023)
CPU x86_64 (x86_64)
Bug reports: https://bugs.chromium.org/p/gerrit/issues/entry?template=Repo+tool+issue
```

## 切换Python 3 版本

查看当前Python版本
```
python -V
```
若返回的版本号为Python3版本，则无需再切换Python版本。若为Python2版本或未发现python，则可以用以下方式切换：

```
#查看当前系统安装的Python版本有哪些
ls /usr/bin/python*

#将python链接到python3
sudo ln -sf /usr/bin/python3 /usr/bin/python

#重新查看默认Python版本
python -V
```
此时系统默认Python版本切换为python3

## 拉取源码

目前野火鲁班猫RK系列板卡使用的SDK主要是两种，分别是通用SDK和专用SDK。自2024年7月开始，新型号的RK系列板卡全面使用通用SDK，旧型号的板卡将逐步切换到通用SDK，其专用SDK也将进入预存档期，只进行BUG修复，不再添加新功能。

以下是专用SDK和通用SDK的对比：

-   稳定性：由于专用SDK已使用的时间长，修复用户反馈较多，相较于新版本的通用SDK稳定性更好，功能更完整。
-   功能性：通用SDK使用了更新的内核版本，其余源码也更新，有更多新特性
-   便利性：对于专用SDK来说，一个SOC型号对应一份SDK，SDK在编译中会产生大量缓存数据占据硬盘空间，如果同时使用多个SOC，就需要拉取多份SDK保存在本地，会占用多份空间，而这里面的大部分内容都是相同的，也不得不存储多份。通用SDK就很好的解决了这一问题，一份SDK就可以编译多个不同型号SOC的板卡，有效节省了编译空间。
-   通用性：由于通用SDK中除了板级配置有差异，其他内容基本都是相同的，不同的位置也做了通用性处理，这就使得我们在开发过程中，一次修改可以应用到所有板卡中，大大减少了重复移植的过程。
-   可维护性：由于LubanCat系列中子系列日益增加，通用SDK同时开发多个soc的特点，有效提升了产品的可维护性，做到对旧产品的超长期维护。
-   构建命令：通用SDK与专用SDK编译命令基本一致，并且在通用SDK中增加了一些更加便利的命令，如使用make kconfig修改内核配置文件，开发更便利。

### 拉取通用SDK

```
#github地址(用户使用)
repo --trace init --depth=1 -u https://github.com/LubanCat/manifests.git -b linux -m lubancat_linux_generic.xml

#内部地址(内部开发使用)
repo init -u git@gitlab.ebf.local:rockchip/linux/manifests.git -b linux -m lubancat_linux_generic.xml

#如果运行以上命令失败，提示：fatal: Cannot get https://gerrit.googlesource.com/git-repo/clone.bundle 
#则可以在以上命令中添加 --repo-url https://mirrors.tuna.tsinghua.edu.cn/git/git-repo 
#例如：

repo --trace init --depth=1 --repo-url https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -u https://github.com/LubanCat/manifests.git -b linux -m lubancat_linux_generic.xml
```


### 拉取专用SDK

以rk356x为例，如果使用rk3588处理器，将下面命令中的rk356x替换为rk3588

```
#github地址(用户使用)
repo --trace init --depth=1 -u https://github.com/LubanCat/manifests.git -b linux -m rk356x_linux_release.xml

#内部地址(内部开发使用)
repo init -u git@gitlab.ebf.local:rockchip/linux/manifests.git -b linux -m rk356x_linux_release.xml

#如果运行以上命令失败，提示：fatal: Cannot get https://gerrit.googlesource.com/git-repo/clone.bundle 
#则可以在以上命令中添加 --repo-url https://mirrors.tuna.tsinghua.edu.cn/git/git-repo 
#例如：

repo --trace init --depth=1 --repo-url https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -u https://github.com/LubanCat/manifests.git -b linux -m rk356x_linux_release.xml 
```

## 更新同步源码

```
.repo/repo/repo --trace sync -c -j4

# 使用 --depth=1 拉取源码后，大部分仓库同步后仅有最新的一次提交，可以使用以下命令来获取完整的仓库
# 进入git仓库内，例如kernel
cd kernel
# 获取整个仓库内容
git fetch --unshallow

```

--depth=1 可以在拉取时进行浅克隆，只拉取最新的一次提交，可以有效减少从网络拉取的内容。如果想拉取完整的带所有提交的内容，可以删除此选项。

# 构建板卡通用镜像

## 安装构建根文件系统依赖
```
sudo dpkg -i debian/ubuntu-build-service/packages/*
sudo apt-get install -f
```

## 一键构建

```
# 选择要构建的板卡的处理器系列
./build.sh chip

# 选择要构建的板卡的配置文件
./build.sh lunch

# 输入对应板卡不同系统配置文件前的序号
Which would you like? [0]:2

# 一键编译
./build.sh
```

## 单独编译

```
# U-Boot 编译
./build.sh uboot

# Kernel 编译
/build.sh kerneldeb
/build.sh extboot

# Debian 编译
./build.sh debian

# 打包update.img镜像
./build.sh updateimg
```


#### 注意

- 指定的配置文件需要与操作系统对应
- 各操作系统只有rootfs的构建不同，U-Boot、Kernel都是相同的
- 生成的镜像保存在 rockdev目录下
- Ubuntu镜像需单独操作
- 镜像详细构建流程请查看在线文档《[野火]嵌入式Linux镜像构建与部署—基于LubanCat-RK系列板卡》 https://doc.embedfire.com/linux/rk356x/build_and_deploy/zh/latest/index.html

## 常见问题

### 第一次拉取使用浅克隆，当前提交与仓库分支对应关系丢失，导致无法更新

如果只需要当前SDK的最新一次提交，建议删除当前SDK重新拉取。

如果本地有代码修改，建议在所在仓库创建新分支提交修改后按以下内容操作，以下操作会拉取所有仓库的所有提交历史，需要从网络获取较大文件。

#### 错误提示1:
```
dev120:~/LubanCat_Linux_SDK$ repo sync -c

... A new version of repo (2.45) is available.
... You should upgrade soon:
    cp /home/jiawen/LubanCat_Linux_SDK/.repo/repo/repo /home/jiawen/bin/repo

fatal: 无效的上游 '2d91fecfd15e5a2c84297bbb1a06fe63e9f69a9c^1'

================================================================================
Repo command failed: UpdateManifestError
```
删除repo配置文件中的浅克隆配置：SDK目录下.repo/manifests.git/.repo_config.json
找到文件中的下面三行删除
```
  "repo.depth": [
    "1"
  ],
```
然后在SDK目录下运行.repo/repo/repo sync -c更新


#### 错误提示2:

```
dev120:~/LubanCat_Linux_SDK$ repo sync -c

... A new version of repo (2.45) is available.
... You should upgrade soon:
    cp /home/jiawen/LubanCat_Linux_SDK/.repo/repo/repo /home/jiawen/bin/repo

.repo/manifests/: discarding 56 commits removed from upstream
首先，回退头指针以便在其上重放您的工作...
应用：test
使用索引来重建一个（三方合并的）基础目录树...
M       rk356x_linux_release.xml
回落到基础版本上打补丁及进行三方合并...
自动合并 rk356x_linux_release.xml
冲突（内容）：合并冲突于 rk356x_linux_release.xml
打补丁失败于 0001 test
Resolve all conflicts manually, mark them as resolved with
"git add/rm <conflicted_files>", then run "git rebase --continue".
You can instead skip this commit: run "git rebase --skip".
To abort and get back to the state before "git rebase", run "git rebase --abort".
error: 无法合并变更。
提示：用 'git am --show-current-patch' 命令查看失败的补丁

================================================================================
Repo command failed: UpdateManifestError
```
删除repo配置文件中的浅克隆配置：SDK目录下.repo/manifests.git/.repo_config.json
找到文件中的下面三行删除
```
  "repo.depth": [
    "1"
  ],
```
进入manifests仓库内，重置git rebase合并状态
```
cd .repo/manifests && git rebase --abort
```
然后在SDK目录下运行.repo/repo/repo sync -c更新


