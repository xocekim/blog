---
title: FreeBSD autologin to X/XFCE/i3/etc
date: 2020-01-29 10:24:00 Z
categories:
- freebsd
tags:
- freebsd
---

I don't like the idea of having a display manager light xdm or lightdm as I am the only user of my PC so here is how I autologin.  

Edit /etc/gettytab and modify the X entry so it reads  
```
    ...
    X|Xwindow|X Window System:\
        :al=USERNAME:\
        :fd@:nd@:cd@:rw:sp#9600:
    ...
```
Edit /etc/ttys to read  
```
...
ttyv8    "/usr/libexec/getty X"    xterm    on    secure
...
```
Now append this to your ~/.profile (assuming your are using bash as your default shell)  
```
...
if [ `/usr/bin/tty` = `/dev/ttyv8` ]; then
    /usr/local/bin/startx
fi
```
Finally create a ~/.xinitrc with your chosen desktop manager executable
```
exec startxfce4
```
