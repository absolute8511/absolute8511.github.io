---
layout: post
title: phpMyadmin连接 mysql问题总结
date: 2008-09-03
categories: tech
tags: [mysql,php]
---

最近弄起了apache＋php＋mysql来玩玩，也下了个phpMyadmin来操作数据库，使用过程中也出了不少问题，现总结如下：

下好phpmyadmin 3.0 后，解压到网站的根目录下这样就可以直接访问了。然后我直接通过

http://localhost/phpMyAdmin-3.0.0-beta-all-languages/

访问。

### 问题1: 出现无法加载mysql扩展的错误。

解决方法：

确保在php.ini文件里面已经将extension=php_mysql.dll，extension=php_mysqli.dll前面的分号去掉了；

而且扩展的目录也修改成正确的目录了，如：extension_dir = "E:\soft\PHP\ext"；

再检查一下ext目录下确实存在相应的扩展文件；

最后还不行的话，就需要将php目录下的libmysql.dll文件拷到system32目录下，以上完成后重启一下服务器。

至此问题应该解决了。

### 问题2: 连接mysql被拒绝

解决方法：

基本是配置文件的原因。解压完phpmyadmin后，需要将libraries目录下的config.default.php文件拷贝到phpmyadmin根目录下并重命名为config.inc.php。修改该文件如下内容：

$cfg['Servers'][$i]['extension'] = 'mysqli';

$cfg['Servers'][$i]['auth_type'] = 'config';            //认证方式，本机调试用此模式，将会用下面配置的用户名和密码登录mysql。

$cfg['Servers'][$i]['user'] = 'root';

$cfg['Servers'][$i]['password'] = 'root用户密码';

至此，问题应该得到解决了。

### 问题3: 没有发现 PHP 的扩展设置mbstring, 而当前系统好像在使用宽字符集以及类似问题无法载入mcrypt扩展,请检查PHP配置等有关扩展的问题。

解决方法：

php的扩展设置问题，首先确保扩展目录下存在相应的dll文件，然后将php.INI文件里面相应的extension=php_mbstring.dll前面的分号去掉，最后重启apache即可。

其它的扩展问题可以用类似方法解决。mcrypt扩展先将相关extension的分号去掉，还不行的话就把php目录下的libmcrypt.DLL文件拷贝到system32目录下，接着重启服务器即可，必要时可能要重启计算机。