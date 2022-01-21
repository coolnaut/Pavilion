## 1. 背景

在工作时，不管是使用gitlab还是公司自己基于git建立的代码托管系统，我们都会拥有一个使用公司邮箱注册一个github账号。但我们总归需要有自己的‘私人空间’，比如自己写一些demo，或者作为云盘等。此时就需要一台电脑管理 2个及以上的github账号。

## 2. 思路

ssh连接方式小使用到RSA技术，如果我们没有改动过ssh的默认目录，那么我们会在~/.ssh目录下看到两个秘钥和一个文件

> id_rsa：私钥
>
> id_rsa.pub：公钥
>
> known_hosts：远程服务器ip及公钥，用于验证服务器公钥是否更改

此时好需要一个配置文件，git在解析的时候会读取配置文件，从而获取相应账号的秘钥，具体配置稍后说明。

具体步骤：

1）生成不同名称的秘钥

2）更改配置文件

3）取消全局用户名

## 3. 实操

ssh-keygen：生成、管理和转换认证密钥

常用的可选参数

> -C comment
>          提供一个新注释，即备注
>
> -c      要求修改私钥和公钥文件中的注释。本选项只支持 RSA1 密钥。
>          程序将提示输入私钥文件名、密语(如果存在)、新注释。
>
>  -f filename
>          指定密钥文件名。
>
>  -T output_file
>          测试 Diffie-Hellman group exchange 候选素数(由 -G 选项生成)的安全性。
>
>  -t type
>          指定要创建的密钥类型。可以使用："rsa1"(SSH-1) "rsa"(SSH-2) "dsa"(SSH-2)

（1）在.ssh目录下生成自定义名称的秘钥

`ssh-keygen -t rsa -f ~/.ssh/id_rsa_2 -C "yourmail@xxx.com`

复制公钥并添加到GitHub上你账户上，路径为 `setting -> ssh and GPG keys`

（2）在.ssh目录下生成配置文件

如果有config文件直接编辑，否则生成。

config文件配置说明：

> Host ： 标识一个配置区段。

> GlobalKnownHostsFile：指定一个或多个全局认证主机缓存文件，用来缓存通过认证的远程主机的密钥，多个文件用空格分隔。默认缓存文件为：/etc/ssh/ssh_known_hosts, /etc/ssh/ssh_known_hosts2.

> HostName：指定远程主机名，可以直接使用数字IP地址。如果主机名中包含 ‘%h’ ，则实际使用时会被命令行中的主机名替换。

> IdentityFile：指定密钥认证使用的私钥文件路径。默认为 ~/.ssh/id_dsa, ~/.ssh/id_ecdsa, ~/.ssh/id_ed25519 或 ~/.ssh/id_rsa 中的一个。文件名称可以使用以下转义符：'%d' 本地用户目录
> '%u' 本地用户名称
> '%l' 本地主机名
> '%h' 远程主机名
> '%r' 远程用户名
>
> 可以指定多个密钥文件，在连接的过程中会依次尝试这些密钥文件。

> Port：指定远程主机端口号，默认为 22 。

> User：指定登录用户名。

> UserKnownHostsFile：指定一个或多个用户认证主机缓存文件，用来缓存通过认证的远程主机的密钥，多个文件用空格分隔。默认缓存文件为： ~/.ssh/known_hosts, ~/.ssh/known_hosts2.

```bash
# default                                                                       
Host default
HostName ssh.github.com
User userName
IdentityFile ~/.ssh/id_rsa
# two                                                                           
Host two
HostName ssh.github.com
User userName
IdentityFile ~/.ssh/id_rsa_2
```

（3）测试

```bash
# ssh -T git@host
ssh -T git@default
ssh -T git@two
# Hi IEIT! You've successfully authenticated, but GitHub does not provide shell access.
```

如果你遇到了如下的错误

```
git@github.com: Permission denied (publickey).
```

你可以尝试将新生成的秘钥需要加入到缓存：

```
 ssh-add -K ~/.ssh/id_rsa_xx
```

到这里就可以克隆远端的仓库了。并且此时可以使用sourceTree进行配置，后面的操作可以跳过

（4）删除全局配置

```bash
git config --global --unset 'user.name'
git config --global --unset 'user.email'
```

在自己的仓库下重新配置

```bash
git config user.name `xxx`
git config user.email `xxx@xxx.com`
```

（5）推送到远端

重新设置远端

```
# host 配置文件的host
# repo 仓库名
git remote add origin git@host:repo
```

```
git push origin master
```

之后再添加远程仓库的时候,就不能直接使用`http`的方式了,只能使用`ssh`方式.