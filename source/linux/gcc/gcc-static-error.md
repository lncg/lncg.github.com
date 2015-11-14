# Centos 无法静态编译链接程序
静态编译链接程序依赖 glibc 的静态库, 但 Centos 在默认情况下是没有安装静态 glibc 库的.

因此在静态编译链接程序时会有如下报错:
```text
# gcc test.c -static
/usr/bin/ld: cannot find -lc
collect2: ld returned 1 exit status
```

然后 `yum install -y glib-static` 即可解决该问题.
