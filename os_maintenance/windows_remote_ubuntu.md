# Windows 远程登陆 Ubuntu
对于ubuntu12.04及之前的版本，直接安装xrdp即可

    $ sudo apt-get install xrdp
    该工具实现了Windows远程桌面的xrpd协议

对于ubuntu14.04及之后的版本，据说由于unity的问题，无法和xrpd实现兼容。因此，考虑使用ubuntu自带的【桌面共享】。

对于ubuntu16.04及之后的版本，由于ubuntu自带的【桌面共享】默认对通讯加密了，而xrdp对此不兼容，因此，要关闭【桌面共享】的通讯加密

    $ gsettings set org.gnome.Vino require-encryption false
    $ reboot

启动ubuntu自带的【桌面共享】
![【桌面共享】](../resoource/desktop-share.PNG)

启动Windows远程桌面连接->选着console
![远程桌面](../resoource/remote-desktop.PNG)