---
title: 记一次git命令故障
date: 2020-07-14 14:22:18
tags: [debug,git]
---
### 现象
&emsp;&emsp;执行 **git status** 命令过于缓慢，比较极端的情况下要20秒左右才有反应。同时会报出以下提示
```
[vagrant@10 sprite]$ git status
On branch master
Your branch is up to date with 'origin/master'.


It took 2.54 seconds to enumerate untracked files. 'status -uno'
may speed it up, but you have to be careful not to forget to add
new files yourself (see 'git help status').
nothing to commit, working tree clean
```
<!--more-->
&emsp;&emsp;我先去搜索了报出信息的答案。git花了大约2.54秒去枚举项目中未被追踪的文件，同时建议我使用 -uno 选项来放弃显示这些未被追踪的文件。

&emsp;&emsp;so，根据这下意识的翻译，我开始寻找这个问题的解决办法。

## 排查步骤

&emsp;&emsp;首先去搜索相关的问题答案。
- 有人建议说使用git gc 命令来清理本地储存库
  + https://www.jianshu.com/p/cee23fd3a198
  + 然而并不好使
- 爆栈网上的答案更加简单粗暴，让我换硬盘
  + https://stackoverflow.com/questions/28156648/git-enumerates-nonexistent-untracked-files-slowly
  + 我没法因为这个就去换硬盘，而且成本也忒高。
- 另外一个问题下，有人说自己提交的时候卡住了。这时候有人建议说低版本git确实容易出现这种情况，升级到新版就不会有这种问题了
  + https://segmentfault.com/q/1010000011913177
  + 我检查了以下自己的版本，因为是yum安装的，确实就是1.8的老版本，于是我接下来去编译安装了2.27新版git
    ```
    [vagrant@10 sprite]$ git --version
    git version 2.27.0
    [vagrant@10 sprite]$ 
    ```
&emsp;&emsp;可是问题并没有解决。
难道真的要换硬盘，我觉定先测试一下读写速度，毕竟我也是ssd，不应该这么慢。这个时候我突然意识到，我的项目是放在vagrant的共享目录里面的，它连接着我的虚拟机和宿主机。

&emsp;&emsp;那么是不是因为共享目录读写速度过慢导致的问题呢？

&emsp;&emsp;于是我去宿主机上执行git status。

&emsp;&emsp;非常快，没有卡顿。
- 我觉得就是这个原因。于是去搜索vagrant共享目录卡顿的内容。
找到了这个
  + https://blog.csdn.net/weixin_43160833/article/details/84073149
  + vagrant建议我装一个插件使用nfs来共享目录
    ```
    $ vagrant plugin install vagrant-winnfsd
    Installing the 'vagrant-winnfsd' plugin. This can take a few minutes...


    Installed the plugin 'vagrant-winnfsd (1.4.0)'!
    ```
  + 修改vagrantfile并且重启试试
    ```
    config.vm.synced_folder "./", "/vagrant", type: "nfs"
    ```
  
  + 报错
    ```
    a host-only network to the machine (with either DHCP or a static IP) for NFS to work.
    ```
  + 原来是忘记配置 private_network
  + 在 Vagrantfile 文件中加上
    ```
    config.vm.network "private_network", ip: "192.168.33.10"
    ```

  + 加上后重启
  + 好了，这回速度终于还算可以了

## 总结
&emsp;&emsp;总的来说问题解决，但是还有一些小小的瑕疵。当我用visualbox保存状态关机，下回启动的时候因为未知的原因，共享目录无法访问了，我猜测是因为网络环境变化引起的。

&emsp;&emsp;必须要正常关机后 vagrant up，这才能继续使用。

&emsp;&emsp;有时间再继续探究原因。

&emsp;&emsp;工欲善其事必先利其器！