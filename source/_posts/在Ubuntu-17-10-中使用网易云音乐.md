---
title: 在Ubuntu 17.10 中使用网易云音乐
date: 2018-02-04 20:59:02
tags:
  - Ubuntu
  - 网易云音乐
---

网易云音乐 for Ubuntu 16.04 1.1.0版在Ubuntu 17.10上运行会有问题，表现是双击运行后GUI一直没有出现，但在进程列表中可以看到其进程并没有结束。这大概是因为运行过程中出了一些问题。因此，我尝试在终端中运行，希望能看到运行的输出，结果如下：

```bash
$ netease-cloud-music
Gtk-Message: Failed to load module "overlay-scrollbar"
[0204/210942.299599:ERROR:nss_util.cc(802)] After loading Root Certs, loaded==false: NSS error code: -8018
```

说句题外话，像这样运行过程中遇到错误，既没有明显的报错信息，让人忍不住双击了好几下；也没有退出，一直占用着系统资源，这种行为是应该抵制的。以后编程时也要注意合理地设计出错时的行为，在出现致命错误时应当给出明显的错误信息，并且及时退出释放资源。

言归正传，看到了错误信息以后，就是使用搜索引擎了。我在[贴吧](http://tieba.baidu.com/p/5453477038)中找到了线索：加上`sudo`就能正常运行。运行结果是这样的：

```bash
$ sudo netease-cloud-music
[sudo] jimmy 的密码： 
[0204/211703.921107:ERROR:nss_util.cc(802)] After loading Root Certs, loaded==false: NSS error code: -8018
QStandardPaths: XDG_RUNTIME_DIR not set, defaulting to '/tmp/runtime-root'

(netease-cloud-music:25107): IBUS-WARNING **: The owner of /home/jimmy/.config/ibus/bus is not root!
# 此处出现GUI
```

报错归报错，但终于其他功能都能跑起来了。

不过每次都从终端中运行也不是个事，最好还是像之前那样，双击以后输入密码，然后就能正常运行。

首先想到的是，在快捷方式的命令中加上`sudo`:

![](/images/20180204212032.png)

双击运行后，发现什么都没有发生，没有要求输入密码，进程列表中也没有网易云音乐。

也就是说，现在的问题是如何在.desktop文件中使用`sudo`运行命令。

在askUbuntu上，我发现有人提过[相同的问题](https://askubuntu.com/questions/814471/how-to-run-desktop-icon-from-sudo)。在下面的答案中，我觉得比较好的解决方案是使用`gksu`代替`sudo`。这两个命令的功能十分相似，主要区别是前者是在GUI中输入密码，后者是在命令行中输入密码。`gksu`不是自带的，需要先安装：

```bash
$ sudo apt-get install gksu
```

之后将desktop中的`sudo`改成`gksu`：

![](/images/20180204212143.png)

然后双击运行后会弹出一个模态框让你输入密码（这里截不了图），之后就一切正常了。

**经验总结**：

- 编程时需要注意出错时程序的行为
- 可以使用`gksu`在GUI中请求权限