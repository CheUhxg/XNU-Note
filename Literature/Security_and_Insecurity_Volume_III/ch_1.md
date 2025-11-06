# 身份认证

身份认证就是弄清楚执行某个操作的用户是谁。

在内核的最低级别，UN\*X只能看到用户的id和组，看不到用户名。内核不知道哪些凭证被使用了，以及是否合法。映射uid到用户名至关重要，因为这种映射就是核查凭证的过程，也是身份认证做的事。

## 密码文件（*OS）

UN\*X一直使用简单的密码文件（/etc/paswd），其中包含用户的详细信息和密码。BSD 4.3和macOS采用了该文件，将其重命名为/etc/master.passwd。

这些年发现使用简单的密码文件具有毁灭性影响，所以该文件格式很快被弃用，只有macOS引导系统进入单用户模式时才会使用。

```
# cat /etc/master.passwd
Password:
##
# User Database
#
# Note that this file is consulted directly only when the system is running
# in single-user mode.  At other times this information is provided by
# Open Directory.
#
# See the opendirectoryd(8) man page for additional information about
# Open Directory.
##
nobody:*:-2:-2::0:0:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0::0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1::0:0:System Services:/var/root:/usr/bin/false
```

> macOS上的/etc/passwd

这个文件的用途是将uid和指定为用户名映射。其中root和mobile（iOS）的shell被设置为/bin/sh，但是/bin/sh在iOS上不可用且没有login(1)或sshd(8)用于登录，所以没有坏处。
* 当存在/bin/sh时，/etc/master.passwd将会被getpwent(3)查询。

## setuid和setgid（macOS）

setuid和setgid作为两个权限位，它们可以在可执行文件中被设置，以允许用户立即获得组长/成员身份。

UN\*X的su(1)命令内部调用了setuid(2)，然而接管另一个用户身份是越权操作。

Darwin一直在减少setuid/setgid类似系统调用的数量，只有部分被保存下来。

## 可插拔认证模块（macOS）

可插拔认证模块（PAM）是标准的UN\*X库，旨在提取和模块化UN\*X的身份认证API。这使得它们的扩展超出了/etc/passwd和/etc/group受限的经典模式，开放给第三方和/或外部认证服务。PAM不仅有效**实现了身份认证API功能的挂接**（与日志记录、审计或策略执行器集成在一起），还将**身份认证逻辑从进程中分离，通过文件实现外部配置**。

> macOS的PAM实现存在于Linux。

开发人员只需要通过pam(3)调用PAM API。此时，调用者要与libpam.dylib库链接。

PAM的模块通过配置文件与二进制文件进行匹配。/etc/pam.d记录了每个受支持的二进制文件。

```
# ls /etc/pam.d
authorization        login.term           screensaver_new_ctk
authorization_aks    other                screensaver_new_la
authorization_ctk    passwd               smbd
authorization_la     screensaver          sshd
authorization_lacont screensaver_aks      su
checkpw              screensaver_ctk      sudo
chkpasswd            screensaver_la       sudo_local.template
cups                 screensaver_new
login                screensaver_new_aks
```

![pam](https://bbs.kanxue.com/upload/attach/201801/581423_7ESVQR2FHZZ2KD3.png)

### 函数类

模块库可以导出以下四个类型（函数类，function class）的API：
* auth（认证）API：提供身份认证函数。
* account（账户）API：提供账户策略管理和执行函数。
* session（会话）API：为已认证的用户设置会话。
* password（密码）API：提供凭证管理函数，使用户能够添加、删除或修改凭证。

### 控制标志

下面这些标志会告诉PAM如何处理模块函数的返回码：
* required: 当使用此控制标志时，当验证失败时仍然会继续进行其下的验证过程，它会返回一个错误信息，但是由于它不会由于验证失败而停止继续验证过程，因此用户不会知道是哪个规则项验证失败。
* requisite: 与required 的验证方式大体相似，但是只要某个规则项验证失败则立即结束整个验证过程，并返回一个错误信息。使用此关键字可以防止一些通过暴力猜解密码的攻击，但是由于它会返回信息给用户，因此它也有可能将系统的用户结构信息透露给攻击者。
* sufficient: 只要有此控制标志的一个规则项验证成功，那么 PAM 构架将会立即终止其后所有的验证，并且不论其前面的 required 标志的项没有成功验证，它依然将被忽略然后验证通过。
* binding：只要有此控制标志的一个规则项验证成功，那么 PAM 构架会继续运行剩余语句后返回成功。
* optional: 表明对验证的成功或失败都是可有可无的，所有的都会被忽略。

把前面这些东西合在一起，就是配置文件，每行：一个函数类+一个控制标志+一个要加载的模块名称。该模块通常只在/usr/lib/pam中被搜索。

> 如果希望所有人无需密码执行su，则在其配置文件加入：auth sufficient pam_permit.so
> 反之，如果希望全面禁用su，则加入：auth required pam_deny.so

## opendirectoryd（macOS）

macOS使用的主要PAM是/usr/lib/pam/pam_opendirectory.so.2，该库与公有框架OpenDirectory.framework链接。

守护进程opendirectoryd是系统中所有目录请求的中心。

### 维护权限

/System/Library/OpenDirectory/permissions.plist定义了节点属性的权限，可以用这些权限来保护ShadowHashData之类的敏感属性，防止非root/wheel用户读取它们。该文件的条目是包含节点数组的字典，用于uuid提供的单个权限，以确保将不同用户名映射到同一个uid。

### 数据存储

对于单机配置，大多数数据在/System/Library/OpenDirectory/Configurations/Local.plist中配置，数据本身在/var/db/dslocal/nodes/Default下。
