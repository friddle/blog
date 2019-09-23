---
title: "Anbox-Play-HeartStone"
categories:
  - Tech
tags:
  - Tools
  - HeartStone
  - Anbox
---

为啥不用genymotion。因为还是虚拟机＋只有8G+大厂都没有在Linux平台做模拟。　　

核心用anbox来弄

安装snapd
sudo apt-get install snap

安装anbox
sudo snap install --devmode --beta anbox
因为Deepin的问题。人家没做那个deepin的内核驱动的包。请按这个流程走下

https://github.com/anbox/anbox-modules


安装houdini镜像
(https://www.linuxuprising.com/2018/07/anbox-how-to-install-google-play-store.html)
```
wget https://raw.githubusercontent.com/geeks-r-us/anbox-playstore-installer/master/install-playstore.sh store.sh
chmod 755 store.sh
sudo ./store.sh
```

然后关键问题来了
装好后请用
snapd stop anbox
snpad start anbox
anbox session-manager --single-window

这三个命令不要有任何其他操作。基本再打开炉石应该没问题了