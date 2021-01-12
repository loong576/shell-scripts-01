## 一、专题背景

最近使用了个自动化平台（详见[自动化运维平台Spug测试](https://blog.51cto.com/3241766/2537675)）进行每周的变更，效果很不错，平台将大量重复繁琐的操作通过脚本分发方式标准化自动化了，平台核心是下发到各个服务器的shell脚本，感觉有必要对shell脚本做个总结，所以有了写本专题的想法。本专题将结合运维实际介绍shell脚本的各项用法，预计10篇左右，将包括系统巡检、监控、ftp上传下载、数据库查询、日志清理、时钟同步、定时任务等，里面会涉及shell常用语法、注意事项、调试排错等。

## 二、本文前言

本文是该专题的第一篇。

做运维的都写过脚本，脚本的第一行`#!/bin/bash`大家都很熟悉，今天就具体讲讲这个第一行：

> - 为什么用使用#!/bin/bash
> - 首行不写或者写得有误有什么影响

在具体测试前先介绍下shell和bash

## 三、什么是shell

```bash
shell是一种特殊的交互式工具。它为用户提供了启动程序、管理文件系统中的文件以及运行在Linux系统上的进程的途径。shell的核心是命令行提示符。命令行提示符是shell负责交互的部分。它允许你输入文本命令，然后解释命令，并在内核中执行。
当一个用户登录Linux系统之后，系统初始化程序init就为每一个用户运行一个称为shell(外壳)的程序。shell就是一个命令行解释器，它为用户提供了一个向Linux内核发送请求以便运行程序的界面系统级程序，用户可以用shell来启动、挂起、停止甚至是编写一些程序。 
```

![image-20210111143520721](https://i.loli.net/2021/01/11/OMRw716UDziXkdS.png)

## 四、什么是bash

在Linux系统上，通常有好几种Linux shell可用。不同的shell有不同的特性，有些更利于创建脚本，有些则更利于管理进程。所有Linux发行版默认的交互shell都是bash shell。bash shell由GNU项目开发，被当作标准Unix shell——Bourne shell（以创建者的名字命名）的替代品。bash shell的名称就是针对Bourne shell的拼写所玩的一个文字游戏，称为Bourne again shell。
除了bash shell，还有dash shell、zsh shell、tcsh、ash等。

bash和dash的区别（后面的测试基于二者的区别）：dash shell只是Bourne shell功能的一个子集， bash shell脚本中的有些功能没法在dash shell中使用，如在脚本中dash无法使用双等号（==）来测试两个字符串是否相等，可以利用这个特性区分bash和dash。

## 五、不写第一行可以不

为了说清楚这个问题，先看两个例子

示例一：

```bash
root@ubuntu1604:~# more first.sh 
echo "Hello World"
root@ubuntu1604:~# ll|grep first.sh
-rw-r--r--  1 root root   19 Jan  8 14:19 first.sh
root@ubuntu1604:~# sh first.sh 
Hello World
root@ubuntu1604:~# ./first.sh
-bash: ./first.sh: Permission denied
root@ubuntu1604:~# chmod u+x first.sh
root@ubuntu1604:~# ./first.sh 
Hello World
```

![image-20210108142453618](https://i.loli.net/2021/01/11/t9IPySHvcwjrz41.png)

| 脚本名   | 首行 | 执行方式 | 执行结果 |
| -------- | ---- | -------- | -------- |
| first.sh | 空   | ./       | 成功     |

脚本first.sh只有一行：echo "Hello World"，脚本执行成功，那是不是意味着首行声明可以不需要写呢，或者说首行的声明会给脚本执行造成什么影响？我们再看个示例。

示例二：

```bash
root@ubuntu1604:~# more dash1.sh 
test1=abcdef
test2=abcdef
if [ $test1 == $test2 ]
then
echo "They're the same!"
else
echo "They're different"
fi
root@ubuntu1604:~# ./dash1.sh 
They're the same!
root@ubuntu1604:~# more dash2.sh 
#!/bin/bash
test1=abcdef
test2=abcdef
if [ $test1 == $test2 ]
then
echo "They're the same!"
else
echo "They're different"
fi
root@ubuntu1604:~# ./dash2.sh 
They're the same!
root@ubuntu1604:~# more dash3.sh 
#!/bin/dash
test1=abcdef
test2=abcdef
if [ $test1 == $test2 ]
then
echo "They're the same!"
else
echo "They're different"
fi
root@ubuntu1604:~# ./dash3.sh 
./dash3.sh: 4: [: abcdef: unexpected operator
They're different
root@ubuntu1604:~# more dash4.sh 
#!/bin/sh
test1=abcdef
test2=abcdef
if [ $test1 == $test2 ]
then
echo "They're the same!"
else
echo "They're different"
fi
root@ubuntu1604:~# ./dash4.sh 
./dash4.sh: 4: [: abcdef: unexpected operator
They're different
```

![image-20210108150618076](https://i.loli.net/2021/01/11/KZ6ehFMnVQDCboY.png)

脚本dash1.sh、dash2.sh、dash3.sh和dash4.sh的执行方式相同，内容也相同（除了首行声明外），结果执行结果却大不相同，dash1.sh、dash2.sh执行正常，dash3.sh和dash4.sh执行报错。这个至少证明首行的声明还是起作用的。

到这里大家肯定云里雾里了，即使之前对shell脚本很清楚的童鞋估计现在也被我绕晕了，这就对了，因为我就是这么过来了……

说回正题，我们先梳理下首行空、/bin/bash、/bin/dash和/bin/sh这4种情况的执行情况，大家先看一张表：

| 脚本名   | 首行        | 执行方式 | 执行结果 |
| -------- | ----------- | -------- | -------- |
| dash1.sh | 空          | ./       | 成功     |
| dash2.sh | #!/bin/bash | ./       | 成功     |
| dash3.sh | #!/bin/dash | ./       | 失败     |
| dash4.sh | #!/bin/sh   | ./       | 失败     |

为了解释这个原因，先介绍下默认的交互shell和默认的系统shell

### 1.默认的交互shell

默认的交互shell会在用户登录某个虚拟控制台终端或在GUI中运行终端仿真器时启动，简单讲就是用户使用登陆交互终端如crt、putty等登陆系统的默认shell。

```bash
root@ubuntu1604:~# echo $SHELL
/bin/bash
root@ubuntu1604:~# cat /etc/passwd|grep root
root:x:0:0:root:/root:/bin/bash
```

![image-20210108152549649](https://i.loli.net/2021/01/11/xkwQBPT6ctGoDaR.png)

默认的交互shell为bash

默认的交互shell由配置文件/etc/default/useradd的SHELL参数决定，感兴趣的童鞋可以修改测试下。

### 2.默认的系统shell

默认的系统shell，用于那些需要在启动时使用的系统shell脚本，“sh+脚本名”这种执行方式就是使用的系统shell，后面会有介绍。

```bash
root@ubuntu1604:~# which sh
/bin/sh
root@ubuntu1604:~# cd /bin/
root@ubuntu1604:/bin# ll|grep sh
-rwxr-xr-x  1 root root 1037528 May 16  2017 bash*
-rwxr-xr-x  1 root root  253816 Jun 16  2017 btrfs-show-super*
-rwxr-xr-x  1 root root  154072 Feb 18  2016 dash*
lrwxrwxrwx  1 root root       4 Feb 20  2019 rbash -> bash*
lrwxrwxrwx  1 root root       4 Feb 20  2019 sh -> dash*
lrwxrwxrwx  1 root root       4 Feb 20  2019 sh.distrib -> dash*
lrwxrwxrwx  1 root root       7 Aug 19  2015 static-sh -> busybox*
```

![image-20210108152648738](https://i.loli.net/2021/01/11/GrEQub8MxSnaL9D.png)

默认的系统shell /bin/sh指向/bin/dash，即默认系统shell为dash。

### 3.脚本第一行测试总结

介绍完了默认交互shell和默认系统shell，在对以上两个示例做个总结：

| 脚本名   | 首行        | 执行方式 | 默认交互shell | 默认系统shell | 等效于        | 执行结果 |
| -------- | ----------- | -------- | ------------- | ------------- | ------------- | -------- |
| first.sh | 空          | ./       | bash          | dash          | bash first.sh | 成功     |
| dash1.sh | 空          | ./       | bash          | dash          | bash dash1.sh | 成功     |
| dash2.sh | #!/bin/bash | ./       | bash          | dash          | bash dash1.sh | 成功     |
| dash3.sh | #!/bin/dash | ./       | bash          | dash          | dash dash1.sh | 失败     |
| dash4.sh | #!/bin/sh   | ./       | bash          | dash          | dash dash1.sh | 失败     |

大家现在可能有些头绪了，通过实验可以得出**结论一：**

> - 1.当脚本没有在首行声明shell时，系统会使用默认的交互shell（即文中的bash）执行脚本；
> - 2.当脚本首行有声明shell时，系统会读取对应的shell；

回到之前的问题：不写第一行可以不？

答案是不写首行声明某些时候不影响脚本执行结果，但是为了规范，建议大家最好养成首行就声明shell的习惯，因为首行 #后面的惊叹号会告诉shell用哪个shell来运行脚本，并且声明只能在首行。

在通常的shell脚本中，井号（ # ）用作注释行。shell并不会处理shell脚本中的注释行。然而，shell脚本文件的第一行是个例外， # 后面的惊叹号会告诉shell用哪个shell来运行脚本。

## 六、执行脚本的两种方式

shell脚本执行有两种方式，“sh+脚本名”和“./脚本名”，这两种方式有啥却别呢？首行如果声明有误或者在第二行声明shell有什么差别没？看看下面的例子：

**示例三：**

```bash
root@ubuntu1604:~# more sh1.sh 
echo "Hello,I am loong576!"
root@ubuntu1604:~# sh sh1.sh 
Hello,I am loong576!
root@ubuntu1604:~# ./sh1.sh 
Hello,I am loong576!
root@ubuntu1604:~# more sh2.sh 
#!/bin/bash
echo "Hello,I am loong576!"
root@ubuntu1604:~# sh sh2.sh 
Hello,I am loong576!
root@ubuntu1604:~# ./sh2.sh 
Hello,I am loong576!
root@ubuntu1604:~# more sh3.sh 
#!/bin/dash
echo "Hello,I am loong576!"
root@ubuntu1604:~# sh sh3.sh 
Hello,I am loong576!
root@ubuntu1604:~# ./sh3.sh 
Hello,I am loong576!
root@ubuntu1604:~# more sh4.sh 
#!/bin/errorsh
echo "Hello,I am loong576!"
root@ubuntu1604:~# sh sh4.sh 
Hello,I am loong576!
root@ubuntu1604:~# ./sh4.sh 
-bash: ./sh4.sh: /bin/errorsh: bad interpreter: No such file or directory
root@ubuntu1604:~# more sh5.sh 

#!/bin/errorsh
echo "Hello,I am loong576!"
root@ubuntu1604:~# sh sh5.sh 
Hello,I am loong576!
root@ubuntu1604:~# ./sh5.sh 
Hello,I am loong576!
```

![image-20210108162623528](https://i.loli.net/2021/01/11/g5EKrSQwVBiC79R.png)

示例三执行汇总：

| 脚本名 | 首行                 | 执行方式 | 默认交互shell | 默认系统shell | 等效于         | 执行结果 |
| ------ | -------------------- | -------- | ------------- | ------------- | -------------- | -------- |
| sh1.sh | 空                   | sh       | bash          | dash          | dash sh1.sh    | 成功     |
| sh1.sh | 空                   | ./       | bash          | dash          | bash sh1.sh    | 成功     |
| sh2.sh | #!/bin/bash          | sh       | bash          | dash          | dash sh2.sh    | 成功     |
| sh2.sh | #!/bin/bash          | ./       | bash          | dash          | bash sh2.sh    | 成功     |
| sh3.sh | #!/bin/dash          | sh       | bash          | dash          | dash sh3.sh    | 成功     |
| sh3.sh | #!/bin/dash          | ./       | bash          | dash          | dash sh3.sh    | 成功     |
| sh4.sh | shell定义有误        | sh       | bash          | dash          | dash sh4.sh    | 成功     |
| sh4.sh | shell定义有误        | ./       | bash          | dash          | errorsh sh4.sh | 失败     |
| sh5.sh | 空，第二行定义且有误 | sh       | bash          | dash          | dash sh5.sh    | 成功     |
| sh5.sh | 空，第二行定义且有误 | ./       | bash          | dash          | bash sh5.sh    | 成功     |

以上实验验证了之前的结论一，也可得出shell声明须在首行的结论；同时从实验也可对比sh和./两种方式的区别得出**结论二：**

> - 1.使用sh方式时，即使无论脚本首行指定的shell是什么或者为空都不影响脚本执行时使用的shell，具体讲在本文“sh+脚本”方式等价于“dash+脚本”；
> - 2.使用./执行脚本时会读取脚本开头指定的shell，若首行未指定shell则使用默认的交互shell，即本文的bash；

当然，二者还有个小区别是sh可以直接运行，./方式需要脚本有执行权限。

## 七、本文总结

本文围绕脚本第一行展开测试，得出首行声明有必要且必须在首行的结论，同时也扩展对脚本的两种执行方式进行了对比，除了测试，也对shell和bash进行了理论介绍。

好了，脚本第一行写好了，下面可以编写各个功能的shell脚本了，敬请期待下篇，大家快上车，关注不迷路，哈哈……

&nbsp;

更多请关注：[shell专题](https://blog.51cto.com/3241766/category18.html)
