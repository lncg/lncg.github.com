# 修改su命令源码: 通过命令行参数输入密码

有个奇葩需求: 需要不停的切换用户. 因为 `su` 命令是交互式输入密码的, 所以不能直接达到自动化需求, 因此有了以下几种解决方案:

1. 用 **root** 用户操作: 悲催, 没有 **root** 权限;
2. `shell expect` 命令: 能满足功能需求, 但在后台调用时可能引发内核崩溃(`n_tty_read+0x2c9/0x970`);
3. `python fabric`: 想想就觉得复杂, 并且它同样可能会遇到第二种方案的问题, 因此没有尝试;

综上, 最终只剩下一种可行的解决方案了: 修改 `su` 命令, 让其在命令行可接受密码参数.

首先, 需要确定 `su` 命令包含在哪个 rpm 包里面:

- 确定 `su` 命令文件路径: `which su`, 得知其位于 `/bin/su`;
- 确定该文件属于哪个 rpm 包: `rpm -qf /bin/su`, 得知其属于 `coreutils-8.4-16.el6.x86_64`;

然后, 去 [pkgs.org](http://pkgs.org/) 搜索下载该 rpm 包对应的源码包.
不一定必须要版本完全相同, 大版本相同或类似即可, 例如[这个就行](http://pkgs.org/centos-6/centos-x86_64/coreutils-8.4-37.el6.x86_64.rpm.html).

下载解压里面的源码压缩包, 并 `./configure && make` 一下,
初始化代码编译依赖环境.

如果一切顺利, `make` 成功后, 就可以修改 `su` 命令源码了.
其实这项工作也很简单, 只要改几个地方就行了...
1. 通过命令行传递密码参数: 默认第一个参数是用户名, 第二个参数是密码,
其他保持不变. 这样只需要在之前获取用户名的地方 `new_user = argv[optind++]`
加一条语句 `passwd = argv[optind++]` 即可.
2. 不再通过交互等待输入密码: 修改 `correct_password()` 函数, 添加密码参数,
注释掉输出交互提示 `Password:` 和获取输入的代码即可.


patch:
```c
diff --git a/su_old.c b/su.c
index f8f5b61..ec64fe7 100755
--- a/su_old.c
+++ b/su.c
@@ -206,9 +206,9 @@ log_su (struct passwd const *pw, bool successful)
    or if PW has an empty password.  */

 static bool
-correct_password (const struct passwd *pw)
+correct_password (char *unencrypted, const struct passwd *pw)
 {
-  char *unencrypted, *encrypted, *correct;
+  char *encrypted, *correct;
 #if HAVE_GETSPNAM && HAVE_STRUCT_SPWD_SP_PWDP
   /* Shadow passwd stuff for SVR3 and maybe other systems.  */
   struct spwd *sp = getspnam (pw->pw_name);
@@ -223,7 +223,7 @@ correct_password (const struct passwd *pw)
   if (getuid () == 0 || !correct || correct[0] == '\0')
     return true;

-  unencrypted = getpass (_("Password:"));
+  /* unencrypted = getpass (_("Password:")); */
   if (!unencrypted)
     {
       error (0, 0, _("getpass: cannot open /dev/tty"));
@@ -393,6 +393,7 @@ main (int argc, char **argv)
 {
   int optc;
   const char *new_user = DEFAULT_USER;
+  char *passwd = NULL;
   char *command = NULL;
   char *shell = NULL;
   struct passwd *pw;
@@ -450,8 +451,10 @@ main (int argc, char **argv)
       simulate_login = true;
       ++optind;
     }
-  if (optind < argc)
+  if (optind < argc) {
     new_user = argv[optind++];
+    passwd = argv[optind++];
+  }

   pw = getpwnam (new_user);
   if (! (pw && pw->pw_name && pw->pw_name[0] && pw->pw_dir && pw->pw_dir[0]
@@ -474,7 +477,7 @@ main (int argc, char **argv)
                           : DEFAULT_SHELL);
   endpwent ();

-  if (!correct_password (pw))
+  if (!correct_password (passwd, pw))
     {
 #ifdef SYSLOG_FAILURE
       log_su (pw, false);
```

修改后可以单独编译 `su` 命令, 而不是编译整个项目: `gcc -o su su.c version.c -L../lib -I../lib -lcoreutils -lcrypt`

然而, 如果用非 root 用户执行刚刚编译的 `su` 命令时, 即使输入正确的密码, 也会提示密码错误的.
这是因为现在 linux 用户密码都存在 **/etc/shadow** 文件中, 需要通过 `getspnam()`
获取密码数据, 然而该函数只允许 **root** 用户调用,
所以需要将它的用户和用户组都设置为 **root**, 并且再设置 `s` 权限位:

```shell
sudo chown root:root ./su
sudo chmod +s ./su
```

OK, 试试看吧~~~


参考链接:

- [重编译Linux命令源代码](http://blog.csdn.net/endoresu/article/details/6967435)
