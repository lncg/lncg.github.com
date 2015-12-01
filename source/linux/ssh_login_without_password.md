# 配置SSH证书免密码登陆服务器

用了这么多年 linux 了, 还在为这个而迷惑, 真实汗颜.

## 服务端是 **linux**, 客户端是 **windows**

通过 putty 提供的 **PuTTYgen** 生成公钥和私钥, 并将它们都保存下来.

此时的公钥和私钥都不是 OpenSSH 标准格式, 都是无法直接使用的.

首先是私钥, 私钥要通过 **PuTTYgen** 的转换功能转换 OpenSSH 格式:
点击菜单栏的 `Conversions` --> `Export OpenSSH Key`, 然后保存到文件.

然后是公钥, 保存到文件的公钥格式是:
```text
---- BEGIN SSH2 PUBLIC KEY ----
Comment: "rsa-key-99991231"
XXXXXXXXXXXX
---- END SSH2 PUBLIC KEY ----
```
如果您查看 linux 下 `~/.ssh/id_rsa.pub`, 您会发现它的格式是 `ssh-rsa XXXXXXXXXXXX username@hostname`.

此时, 可以手动转换一下格式, 也可以直接复制 **PuTTYgen** 窗口提示内展示内容(界面有提示: `Public key for pasting into OpenSSH authorized_keys file`),
其格式为: `ssh-rsa XXXXXXXXXXXX rsa-key-99991231`,
将其追加到 `~/.ssh/authorized_keys` 文件即可.


最后需要检查一下服务端的 SSHD 服务配置文件: `/etc/ssh/sshd_config`,
确保允许使用成对的秘钥系统进行登录:
```
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile  .ssh/authorized_keys

PasswordAuthentication no // 禁用密码登录, 已经启用证书登录了, 最好禁掉它
```

然后重启 SSH 服务, 尝试登录以下吧~


## 服务端和客户端都是 **linux**
那就省事了, 也不用使用 **PuTTYgen** 生成并转化公钥/私钥了,
直接在客户端通过 `ssh-keygen` 命令生成公钥/私钥, 然后将公钥复制到服务端的 `~/.ssh/authorized_keys` 即可, 其他配置同上.



### 参考文章:
- [鳥哥的 Linux 私房菜: 远程联机服务器SSH/XDMCP/VNC/RDP](http://vbird.dic.ksu.edu.tw/linux_server/0310telnetssh_2.php)
- [Manually generating your SSH key in Windows](https://docs.joyent.com/public-cloud/getting-started/ssh-keys/generating-an-ssh-key-manually/manually-generating-your-ssh-key-in-windows)
- [ssh证书登录(实例详解)](http://www.cnblogs.com/ggjucheng/archive/2012/08/19/2646346.html)
