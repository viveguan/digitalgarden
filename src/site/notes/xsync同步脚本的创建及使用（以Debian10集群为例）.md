---
{"dg-publish":true,"permalink":"/xsync-debian10/"}
---

# xsync同步脚本的创建及使用（以Debian10集群为例）
## 1、简介
在集群机器配置时，经常需要将一个文件或目录copy到同样的多台集群上，如果一个一个机器去复制，比较麻烦。如果有一个办法，通过一条命令就可以实现这个目的，就简单多了。[xsync](https://so.csdn.net/so/search?q=xsync&spm=1001.2101.3001.7020)就是这样一个同步脚本。xsync其实是对rsync脚本的二次封装，脚本内容可以根据自己需要进行修改。

## 前置知识
### 2.1 [scp](https://so.csdn.net/so/search?q=scp&spm=1001.2101.3001.7020)与rsync
文件或目录跨机器复制非常常用，一般可以通过scp命令和rsync命令来实现。

* scp ：命令格式为`scp -r $pdir/$fname $user@host:$pdir/$filename`
很好理解，-r表示递归复制，然后给定本机的文件或目录地址 和 目标机器的用户名、主机IP、目的地路径和文件名就可以完成拷贝操作。使用 scp 传输数据时，文件是加密的，因此任何敏感内容都不会在传输过程中被篡改。\* rsync ： 命令格式为 `rsync -av $pdir/$fname $user@$host:$pdir/$fname`
其中命令的含义与scp相似，-a表示归档拷贝，-v表示显示复制过程。

scp与rsync的差别：

1. scp 通过安全的 SSH 连接将文件从本地机器复制到远程机器，而 rsync 允许您同步远程文件夹。
2. scp 读取源文件并将其写入目标，是在本地或通过网络执行线性复制。rsync 也是在本地或通过网络复制文件，但它使用特殊的增量传输算法和一些优化来使操作更快。
3. scp 总是安全的，而 rsync 必须通过 SSH 传输才能安全。
4. 如果要传输大文件，并且传输在完成之前断开连接，rsync 会从中断的地方继续传输，而 scp 没有。
5. rsync 比较每一端的文件并只传输更改文件的更改部分，当你第一次传输文件时，它的行为与 scp 非常相似，但是对于大多数文件未更改的第二次传输，它推送的数据将比 scp 少得多。这也是一种重新启动失败传输的便捷方法，你只需重新发出相同的命令，它就会从上次中断的地方开始，而 scp 将从头开始。

### 2.2 ssh
其中我们可以看到scp与rsync都可以基于ssh进行文件传输，如果不想每次都输入密码，可以配置集群之间的无密登录传输。

ssh会话连接过程：

1. 本地向远程服务端发起连接
2. 服务端随机生成一个字符串发送给发起登录的本地端
3. 本地对该字符串使用私钥（\~/.ssh/id\_rsa）加密发送给服务端
4. 服务端使用公钥（\~/.ssh/id\_rsa.pub）对私钥加密后的字符串进行解密
5. 服务端对比解密后的字符串和第一次发送给客户端未加密的字符串，若一致则判断为登录成功

![image](https://github-cf.viveguan.top/imgs/pn7ZF9UDZaac_k28Psj-Rqb2FAs0rrW9vLPa3FVOXbU.png)

6. 

## 3、配置集群hosts
首先，为啥需要配置集群的hostname呢？因为集群IP不好写。

> hosts文件是linux系统中负责ip地址与域名快速解析的文件，以ASCII格式保存在/etc目录下，Debian的对应文件是/etc/hosts。hosts文件包含了ip地址和主机名之间的映射，包括主机名的别名，在没有域名服务器的情况下，系统上的所有网络程序都通过查询该文件来解析对应于某个主机名的ip地址，否则就需要使用DNS服务程序来解决。通常可以将常用的域名和ip地址映射加入到hosts文件中，实现快速方便的访问。

可以在`/etc/hosts`这个文件中配置集群IP的别名



![image](https://github-cf.viveguan.top/imgs/3pNQA5ptvwTrLcKXVIQm7mOUPfYksoujHFBQ481gE7s.png)




然后可以ping hostname来检验是不是可以ping通。

## 4、配置免密登录
通过上述的ssh的原理介绍可以知道，需要在本地先生成公钥和私钥，然后再将公钥发送到目的机器上，进行ssh操作。

1、进入到用户目录下，使用命令`ssh-keygen`，然后敲三个回车。

```bash
[root@zkos1 ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:kXclvgbcNBCbW9Z88eP1brP1TtPOc+YAuWTw0xi4QrU root@zkos1
The key’s randomart image is:
±–[RSA 2048]----+
| +o+ … |
| + O * o|
| + E B o.+|
| . o X …+|
| S o @ …|
| . + + …|
| . .o|
| BO|
| =O|
±—[SHA256]-----+
123456789101112131415161718192021
```
![image](https://github-cf.viveguan.top/imgs/qhgqYlviDL8uZmRV45z8wU53LtI4BwSQbLndOGErPts.png)




注意，在\~目录下.ssh文件夹里就生成了两个文件id\_rsa（私钥）、id\_rsa.pub（公钥），下一步就是将公钥传给目的机器了。

2、同步到目的机器的目录下，使用命令

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@servername
1
```
其他方法可以参考博客：[传送门](https://blog.csdn.net/nalw2012/article/details/98322637)。

3、意外情况处理。显然，有的集群可能机器之间本来不允许ssh通过密码连接，那么配置时就发送不了公钥到目的机器，此时可以参照下面这篇博客来解决问题。

可以会出现的问题在这片博客中有介绍：解决[Permission denied(publickey)](https://blog.csdn.net/yjk13703623757/article/details/114936739)
注意，如果出现上面的问题，再改了/etc/ssh/sshd\_config配置文件之后需要使用命令`systemctl restart sshd`重启sshd服务

## 5、xsync[脚本编写](https://so.csdn.net/so/search?q=%E8%84%9A%E6%9C%AC%E7%BC%96%E5%86%99&spm=1001.2101.3001.7020)及使用
配置完之后，终于到达了xsync脚本的编写了。

**第一步**：看看rsync这个命令能用不？如果用不了，就安装一下rsync

```bash
apt-get install rsync -y
1
```
如果上述命令失败了，运行下面命令之后再运行上述命令

```Plain Text
apt-get update
apt-get upgrade
12
```
**第二步**：创建脚本xsync

```bash
#!/bin/bash

#1.判断参数个数
if [ $# -lt 1 ]
then
    echo Not Enough Arguement!
    exit;
fi

#2.遍历集群所有机器   请改成你自己的主机映射
for host in host1 host2 host3
do
    echo =============== $host ==================
    #3.遍历所有目录，挨个发送
    for file in $@
    do
        #4.判断文件是否存在
        if [ -e $file ]
            then
            #5.获取父目录
            pdir=$(cd -P $(dirname $file); pwd)
            fname=$(basename $file)
            # 创建文件夹和传输文件。请改成你自己的端口号
            ssh -p 32200 $host "mkdir -p $pdir"
            rsync -av -e 'ssh -p 32200' $pdir/$fname $host:$pdir
        else
            echo $file does not exists!
        fi
    done
done
123456789101112131415161718192021222324252627282930
```
上述代码注释已经很清楚啦，要改主机映射名和端口号。

第三步：赋予脚本执行权限`chmod 777 filename`

第四步：为了使的脚本能够在任何一个文件夹下运行，可以将脚本添加进全局变量中。通过`echo $PATH`查看全局环境变量的路径，将其拷贝到其中一个路径中。

然后就可以开始测试啦，应该是没啥问题。



