---
title: tmux简单操作
date: 2018-04-16
categories: 
- tools
---
> 简单记录一下tmux的应用

# tmux是啥
Tmux 是一个 BSD 协议发布的终端复用软件，用来在服务器端托管同时运行的 Shell。那么 Tmux 用起来是怎样的呢？看图：
![tmux-look](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/tmux-look.png)

# tmux能做啥
你是否曾经开过一大堆的 Terminal？有没有把它们都保存下来的冲动？Tmux 的Session就是做这件事情的！ 你可以随时退出或者进入任何一个 Session。每个 Session 有若干个 Window，每个 Window 又可以分成多个窗格（Pane）。 极大地满足 Terminal 用户的需求。

此外即使 iTerm/Terminal/Konsole 意外关闭也没关系，因为 Session 完全保存在 Tmux Server 中。 再次打开 Terminal 时只需 tmux attach 便可回到你的工作区，就像从未退出过一样。 如果希望重启电脑后仍然生效，你可能需要 动手写脚本 或者 使用插件。

# 安装使用
首先进行安装：
``` php
brew install tmux       # OSX
pacman -S tmux          # archlinux
apt-get install tmux    # Ubuntu
yum install tmux        # Centos
```

## 强烈建议源码安装，使用最新版本 
 
安装好后就可以启用一个Tmux Session了：

``` c
tmux new -s myname (可以指定Session名）
```

# tmux简单操作

在Tmux Session中，通过prefix + $可以重命名当前Session。其中 prefix 指的是tmux的前缀键，所有tmux快捷键都需要先按前缀键。它的默认值是Ctrl+b。

 prefix c可以创建新的窗口（Window）， prefix %水平分割窗口（形成两个Pane）， prefix "垂直分割窗口。退出当前Session的快捷键是 prefix d。然后在Bash中可以查看当前的tmux服务中有哪些Session：

## tmux后台运行
``` php
ctrl d
```

## tmux 唤起
``` java
tmux attach
tmux a -t myname  (or at, or attach)
```

## tmux 分割窗口
``` java
<prefix>c
可以创建新的窗口（Window）
<prefix>%水平分割窗口（形成两个Pane），
<prefix>"垂直分割窗口。
退出当前Session的快捷键是<prefix>d。
```

## tmux 简单配置

``` shell
    //将hjkl替代上下左右
    bind k selectp -U # above (prefix k)
    bind j selectp -D # below (prefix j)
    bind h selectp -L # left (prefix h)
    bind l selectp -R # right (prefix l)

    //C-j，C-k, C-h, C-l跳转pane大小
    bind -r ^k resizep -U 3 # upward (prefix Ctrl+k)
    bind -r ^j resizep -D 2  # downward (prefix Ctrl+j)
    bind -r ^h resizep -L 5 # to the left (prefix Ctrl+h)
    bind -r ^l resizep -R 5 # to the right (prefix Ctrl+l)

    #分割窗口 
    #C-b - 横向分割
    #C-b | 竖向分割
    unbind '"'
    bind - splitw -v
    unbind %
    bind | splitw -h # horizontal split (prefix |)


    #Easy config reload
    bind-key r source-file ~/.tmux.conf \; display-message "tmux.conf reloaded."

    #设置为vi键位
    setw -g mode-keys vi

    #同步操作
    bind-key t setw synchronize-panes on # 打开所有pane之间操作同步
    bind-key T setw synchronize-panes off # 关闭所有pane之间的操作同步

    #设置默认shell
    set-option -g default-shell /bin/zsh

    #List of plugins
    set -g @plugin 'tmux-plugins/tpm'
    set -g @plugin 'tmux-plugins/tmux-sensible'
    set -g @plugin 'tmux-plugins/tmux-resurrect'
    set -g @plugin 'tmux-plugins/tmux-continuum'

    #Enable automatic restore
    set -g @continuum-restore 'on'

    #Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
    run '~/.tmux/plugins/tpm/tpm'
```

