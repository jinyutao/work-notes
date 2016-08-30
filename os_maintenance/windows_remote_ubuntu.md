# Windows 远程登陆 Ubuntu
对于ubuntu12.04及之前的版本，直接安装xrdp即可

    sudo apt-get install xrdp
    该工具实现了Windows远程桌面的xrpd协议

对于ubuntu14.04及之后的版本，据说由于unity的问题，无法和xrpd实现兼容。因此，考虑安装【】

    gsettings set org.gnome.Vino require-encryption false
 and reboot.