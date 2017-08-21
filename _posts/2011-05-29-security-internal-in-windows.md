---
layout: post
title: Ｗindows安全机制内幕
date: 2011-05-29
categories: tech
tags: [windows,security,IntegrityLevel]
comment: True
---

最近研究了点关于Ｗindows安全机制以及COM的 Reg-Free方案。研究的原因是试图构造一个安全的第三方插件的沙箱环境，而第三方插件是实现为COM组件的，因此又涉及到在受限环境下的COM调用的问题，为了解决受限环境的COM注册问题引入了Reg-Free的方案。

整个插件环境是和主程序隔离的，每个插件都使用一个独立进程，插件的宿主进程是一个进程外的COM组件，主进程启动宿主进程后，有宿主进程负责创建相应的插件对象，插件被创建起来后就在宿主进程中运行。为了保证插件不影响用户的系统，需要在宿主进程创建插件对象之前将宿主进程放进一个受限的沙箱环境中。


Ｗindows的安全模型主要由2个部分组成， 一个是用于授权的Ａccess Ｔoken，一个是用于保护安全对象的Ｓecurity Ｄescriptor。一个进程访问某个安全对象时，系统会使用对象的
Ｓecurity Ｄescriptor检查当前进程的Ｔoken是否有权限访问该对象。
Ａccess Ｔoken包含了一些SID用于标示它拥有的安全身份，另外还包含一些Ｐrivilege信息以及其他授权信息。
Ｓecurity Ｄescriptor包含了用于控制访问权限的DACL列表以及用于审核的SACL列表。ACL列表是一系列的ACE信息，每个ACE包含了某个SID的访问权限，当一个进程访问一个对象时，系统会逐个检查对象的ACE项，如果该进程的Ｔoken中包含的SID匹配ACE中的SID,那么系统就会执行ACE中对该SID的访问控制操作（拒绝或者允许），需要注意的是系统是按照列表从头到尾的顺序逐个检测ACE,一旦有拒绝或者允许的匹配项，都会立刻停止检测后继的ACE项。（除非token中有指定ＲestrictedSID）.

下图是2个不同线程访问一个安全对象的过程：
![](/img/windows-security-ace.png)

关于Ｗindows安全机制，它提供了几种机制来确保安全性，主要有ＲestrictedToken，Ｊob Object，Ａlternate Ｄesktop，以及在Ｖista引入的IntegrityLevel的方案来简化和加强 
Ｗindows下的安全隔离。

对于ＲestrictedToken可以使用ＣreateＲestrictedToke函数创建一个受限的Ｔoken，然后使用ＣreateProcessAsUser以受限的token来启动受限进程。而Ｊob可以限制一个进程使用的系统资源，比如CPU使用率，内存使用率，活动进程，剪切版的使用等，在XP下还可以给Ｊob设置安全token来设置job的安全属性（Ｖista以上不支持job的安全属性了）。Ｖista以后引入的IntegrityLevel有四个级别分别为Ｌow, Medium, High, System，低级别的属性不能访问具有高级别的对象，默认所有安全对象都是Ｍedium级别的。另外IntegrityLevel还对注册表的访问和文件的操作做了一些重定向的处理，以保证程序的兼容性。特别是COM调用相关的一些处理，使得即使在最低的级别下一些COM的调用仍然能成功。
由于XP,和XP以上的系统在安全机制上的一些区别，不得不做2种不同的处理。在XP系统下，主要使用Job以及其安全属性，使用JOBOBJECT_EXTENDED_LIMIT_INFORMATION, JOBOBJECT_BASIC_UI_RESTRICTIONS以及JOBOBJECT_SECURITY_LIMIT_INFORMATION设置一个受限的JOB并将进程放入该Job中，同时还要使用AdjustTokenPrivileges将进程的一些Ｐrivilege信息删除。而在Ｖista以后的系统中，无法使用JOBOBJECT_SECURITY_LIMIT_INFORMATION设置Job的安全属性，改用IntegrityLevel将进程的安全级别设置为Ｌow。

在上述的受限环境中，进程的写权限几乎被完全禁止了（除了设定为较低安全级别的对象），仅剩下读权限，这样基本能防止插件对系统造成损害了。
当然，在这种受限环境下,经过测试，XP系统下的COM进程间通信会有一些问题，而Ｖista下的IntegrityLevel似乎对这种情况专门做了处理，因此发现vista下在受限环境下的COM进程间通信一切顺利。为了处理XP下在受限环境下的COM调用和注册，引入一种COM的免注册方案，即Registration-Free 技术。主要就是利用manifest文件让COM的注册脱离注册表依赖。关于Ｒeg-Ｆree技术参考文献有2篇文章写的已经很详细了。需要注意的是，XP和Ｖista之后的manifest文件的都去顺序问题，XP系统是先尝试读取外部manifest文件，没找到才尝试读取dll文件本身嵌入的manifest资源，而vista之后刚好反过来了。另外，进程外的COM组件是不支持Ｒeg-Ｆree的。(经测试，XP下使用reg-free的方案好像只能解决COM进程内的接口调用，对于进程间的接口调用还得另做处理。)

最后，还有一个需要注意的是Ｗin7下的Ｊob处理，由于Ｗin7下的explorer进程会将进程放入一个PCA的Ｊob中，为了避免使用我们自己的Ｊob失败，我们需要防止Ｗin7的这种自作主张的行为，需要在manifest文件中加入以下内容
```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
 <v3:trustInfo xmlns:v3="urn:schemas-microsoft-com:asm.v3">
<v3:security>
 	<v3:requestedPrivileges>
   	<v3:requestedExecutionLevel level="asInvoker" uiAccess="false" />
 	</v3:requestedPrivileges>
</v3:security>
 </v3:trustInfo>
 <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
<application>
 	<!--The ID below indicates application support for Windows Vista -->
 	<supportedOS Id="{e2011457-1546-43c5-a5fe-008deee3d3f0}"/>
 	<!--The ID below indicates application support for Windows 7 -->
 	<supportedOS Id="{35138b9a-5d96-4fbd-8e2d-a2440225f93a}"/>
</application>
 </compatibility>
</assembly>
```

关于Ｗindows安全方面更多详细的资料可以查看以下参考文献。
参考文献：
http://msdn.microsoft.com/en-us/library/aa379570(v=vs.85).aspx

http://www.tenouk.com/ModuleH.html

http://msdn.microsoft.com/en-us/library/aa480244.aspx

http://www.google.co.uk/patents/about?id=m40IAAAAEBAJ

about COM Reg-Free,
http://blogs.msdn.com/b/junfeng/archive/2006/05/17/registration-free-com-net-interop.aspx
http://msdn.microsoft.com/en-us/library/ms973913.aspx
http://msdn.microsoft.com/en-us/magazine/cc188708.aspx

