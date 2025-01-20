---
categories: [大数据, 鉴权]
date: 2020-03-13 00:00:00 +0800
tags: [kerberos]
title: Windows域认证Kerberos详解
---

# Windows域认证Kerberos详解

一直以为Kerberos是HDP平台的组件，直到看了这篇文章，才发现Kerberos和本身weindows域控（Domain Contoller）关系更亲近一些，这里贴一下来自CSDN Sp4rkW的文章。

windows 域在工作中经常遇到，一直没有好好总结过，乘着最近有时间，将自己所理解的与大家分享一下
## 0x01、 Kerberos认证简介

windows 对于身份验证有多种方式，比如现在笔记本很常见的指纹解锁开机。在域中，依旧使用 Kerberos 作为认证手段。

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584080382_94.png)

Kerberos 这个名字来源于希腊神话，是冥界守护神兽的名字。

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584080404_27.png)
Kerberos 是一个三头怪兽，之所以用它来命名一种完全认证协议，是因为整个认证过程涉及到三方：客户端、服务端和 KDC（Key Distribution Center）。在 Windows 域环境中，KDC 的角色由 DC（Domain Controller）来担当。

Kerberos 是一种基于票据 ticket 的认证方式，举个简单的例子来说：

你想参加一场舞会，被拒绝，对方告诉你需要门票
你又去往领门票的地方，对方审查你的资格后，发给你一张门票
最终你凭着这张门票参加了这场舞会，但是对于其他场次的舞会，你任须要去经过资格审查领取门票
下面就来详细聊聊域认证 Kerberos ，值得注意的是，这个流程与上述例子还是有区别的，切勿照搬例子带入

## 0x02、域认证 Kerberos 流程
这块牵涉到的名词比较多，先在前面进行列出，并给出简写
||简写|全拼||
|---|---|---|---|
||DC|Domain Controller 域控||
||KDC|Key Distribution Center 密钥分发中心||
||AD|Account Database 账户数据库||
||AS|Authentication Service 身份验证服务||
||TGS|Ticket Granting Service 票据授与服务||
||TGT|Ticket Granting Ticket 票据中心授予的票据||
先来就下图做一个详细的流程说明

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584080682_7.png)

### 1． Authentication Service Exchange 认证服务交换
通过这步骤，AS 实现对 Client 身份的确认，并颁发给该 Client 一个 TGT 。具体过程如下：

client 向 AS 发送 `AS Request`, 为了确保 `AS Request` 仅限于自己和 KDC 知道，client使用自己的 `Master Key`（用户密码的一种hash） 对 `AS Request` 的主体部分进行加密（KDC 可以通 AD 获得该Client 的 `Master Key`）。

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584080901_8.png)
`AS Request` 的大体包含以下的内容：

- Pre-authentication data：被 Client 的 `Master key`加密过的 Timestamp，用于证明自己是所发送用户名对应的那个用户
- Client name & realm: 简单地说就是Domain name\Client name
- Server Name：注意这里的 `Server Name`并不是 Client 真正要访问的 Server 的名称，而我们也说了 TGT 是和 Server 无关的（ Client 只能使用 Ticket ，而不是 TGT 去访问Server ）。这里的 `Server Name` 实际上是 `KDC TGS 的 Server Name`。

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584081257_68.png)
AS 通过它接收到的 `AS Request` 验证发送方的是否是在 Client name & realm 中声称的那个人，也就是说要验证发送放是否知道 Client 的 Password 。所以 AS 只需从 AD 中提取 Client 对应的 Master Key对 Pre-authentication data 进行解密，如果是一个合法的 Timestamp，则可以证明发送方提供的用户名是存在于白名单中且密码对应正确的。

AS 通过它接收到的 `AS Request` 验证发送方的是否是在 Client name & realm 中声称的那个人，也就是说要验证发送放是否知道 Client 的 Password 。所以 AS 只需从 AD 中提取 Client 对应的 `Master Key`对 Pre-authentication data 进行解密，如果是一个合法的 Timestamp，则可以证明发送方提供的用户名是存在于白名单中且密码对应正确的。
在这里，可能会产生一个疑问，为什么要使用 Timestamp，这种时间戳？

我们试想一下，如果恶意攻击者抓取了一个这样的`AS Request`数据包，用算力一直去爆破尝试，那是不是有不小的机会获取 TGT 。所以在 AS 进行其他操作之前，会先提取`AS Request`中的 Timestamp ，并同当前的时间进行比较，如果他们之间的偏差超出一个可以接受的时间范围（一般是5mins），AS 会直接拒绝该 Client 的请求。当`AS Request`中的 Timestamp 早于上次认证时间点，也是直接拒绝。

验证通过之后，AS 将一份 `AS Response` 发送给 Client。AS Response 主要包含两个部分：本 Client 的 `Master Key` 加密过的 Logon Session Key 和被自己（KDC）加密的TGT。

而TGT大体又包含以下的内容：

- Session Key: SKDC-Client：Logon Session Key
- Client name & realm: 简单地说就是Domain name\Client
- End time: TGT到期的时间。

Client 通过自己的 Master Key 对第一部分解密获得 Logon Session Key之后，携带着 TGT 便可以进入下一步：TGS Exchange。


### 2． Ticket Granting Service Exchange 票据授与服务交换
TGS Exchange 通过 Client 向 KDC 中的 TGS 发送 `TGS Request` 开始。`TGS Request`大体包含以下的内容：

- TGT：Client 通过 AS Exchange 获得的 TGT，TGT 被 KDC 的 `Master Key` 进行加密。
- Authenticator：用以证明当初 TGT 的拥有者是否就是自己， client 端使用 Logon session key 进行加密
- Client name & realm: 简单地说就是 Domain name\Client 。
- Server name & realm: 简单地说就是 Domain name\Server ，这回是 Client 试图访问的那个 Server 。
- 时间戳，同第一步，这里略

Hint ：Authenticator在这里可以理解为仅限于验证双方预先知晓的内容，相当于联络暗号。

![图片描述](/assets/media/Windows域认证Kerberos详解/tapd_48728548_base64_1584082330_67.png)

TGS 收到`TGS Request`在发给 Client 真正的 Ticket 之前，先得确认 Client 提供的那个 TGT 是否是 AS 颁发给它的。于是它得通过 Client 提供的 Authenticator 来证明。但是 Authentication 是通过 Logon Session Key 进行加密的，而自己并没有保存这个 Session Key 。所以 TGS 先得通过自己的 Master Key 对 Client 提供的 TGT 进行解密，从而获得这个Logon Session Key ，再通过这个 Logon Session Key 解密 Authenticator 进行验证。

换句话说，TGS 接收到请求之后，现通过自己的密钥解密 TGT 并获取 Logon Session Key ，然后通过 Logon Session Key 解密 Authenticator ，进而验证了对方的真实身份。

验证通过向对方发送 `TGS Response`。这个`TGS Response`有两部分组成：

- 使用Logon Session Key（SKDC-Client）加密过用于Client和Server的Session Key（SServer-Client）
- 使用Server的Master Key进行加密的Ticket。

该Ticket大体包含以下一些内容：

- Session Key：SServer-Client。
- Client name & realm: 简单地说就是Domain name\Client。
- End time: Ticket的到期时间。

Client 收到`TGS Response`，使用 Logon Session Key，解密第一部分后获得 Session Key （注意区分 Logon Session Key 与 Session Key 分别是什么步骤获得的，及其的区别）。有了 Session Key 和 Ticket ， Client 就可以之间和 Server 进行交互，而无须在通过 KDC 作中间人了。

Client如果使用Ticket和Server怎样进行交互的，这个阶段通过我们的第3个步骤来完成：CS（Client/Server ）Exchange。

### 3． CS（Client/Server ）Exchange
Client 通过 TGS Exchange 获得 Server 的 `Session Key` ，随后创建用于证明自己就是 Ticket 的真正所有者的 Authenticator 和时间戳，并使用 `Session Key` 进行加密。最后将这个被加密过的 Authenticator，时间戳 和 Ticket 作为 Request 数据包发送给 Server 。此外还包含一个 Flag 用于表示 Client 是否需要进行双向验证。

Server 接收到 Request 之后，通过自己的 Master Key 解密 Ticket，从而获得 `Session Key` 。通过 `Session Key` 解密 Authenticator ，进而验证对方的身份。验证成功，让 Client 访问需要访问的资源，否则直接拒绝对方的请求。时间戳的作用，同第一步，这里略。

实际上，到目前为止，服务端已经完成了对客户端的验证，但是，整个认证过程还没有结束。谈到认证，很多人都认为只是服务器对客户端的认证，实际上在大部分场合，我们需要的是双向验证（Mutual Authentication）——访问者和被访问者互相验证对方的身份。现在服务器已经可以确保客户端是它所声称的那么用户，客户端还没有确认它所访问的不是一个钓鱼服务呢。

为了解决客户端对服务器的验证，服务要需要将解密后的 Authenticator 再次用 Session Key 进行加密，并发挥给客户端。客户端再用缓存 Session Key 进行解密，如果和之前的内容完全一样，则可以证明自己正在访问的服务器和自己拥有相同 Session Key，而这个会话秘钥不为外人知晓。

三个流程的总结
上面的文字很多，为了帮助理解，下面我用一些简写去代替上面的名词，粗略的将域认证 Kerberos 原理再次讲述一遍
||简写|含义||
|---|---|---|---|
||A|client||
||B|AS||
||C|TGS||
||D|server||
1. A 访问 B，发送请求包，其中包括 A 的名字和使用A 的 key_a 加密的时间戳
2. B 通过查询内部数据库，得到 请求包中名字对应的 key ，如果用这把 key 可以解出一个规范的时间戳，且该时间戳时间与当前时间戳差距小于5 min，则返回 A 一个返回包
3. 返回包包括使用 a_key 加密的 key1，和 A 无法使用的 data1
4. A 使用 a_key 解密出 key1 ，将联络暗号使用 key1 进行加密和 data1 一起发送给 C
5. C 收到请求包，解开 data1 ，取出 key1。使用 key1 解密联络暗号，确认 A 的身份
6. 确认无误，C 返回 A 一个返回包。包括使用 key_d 加密的 data2 和 key2
7. A 使用 key2 加密联络暗号，将其和 data2 一起发往 D 处
8. D 使用 key_d 解密出 data2 ，取出 key2 ，验证联络暗号
9. D 验证无误后使用 key2 加密联络暗号2发往 A
10. A 使用 key2 解密联络暗号2，确认 D 身份，最终实现资源访问

如果还是不能理解的话，补充一个趣味例子吧

- 用户要去游乐场，首先要在门口检查用户的身份(即 CHECK 用户的 ID)，如果用户通过验证，游乐场的门卫 (AS) 即提供给用户一张门卡 (TGT)。

- 这张卡片的用处就是告诉游乐场的各个场所，用户是通过正门进来，而不是后门偷爬进来的，并且也是获取进入场所一把钥匙。

- 这时摩天轮的服务员拦下用户，向用户要求提供摩天轮的 (ST) 票据，用户说用户只有一个门卡 (TGT), 那用户只要把 TGT 放在一旁的票据授权机 (TGS) 上刷一下。

- 票据授权机 (TGS) 就根据用户现在所在的摩天轮，给用户一张摩天轮的票据 (ST), 这样用户有了摩天轮的票据，现在用户可以畅通无阻的进入摩天轮里游玩了。

- 当然如果用户玩完摩天轮后，想去游乐园的过山车玩耍一下，那用户一样只要带着那张门卡 (TGT). 到过山车的票据授权机 (TGS) 刷一下，得到过山车的票据 (ST) 就可以进入过山车进行玩耍。

- 当用户离开游乐场后，用户的 TGT 就已经过期并且销毁。

————————————————
版权声明：本文为CSDN博主「Sp4rkW」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/wy_97/article/details/87649262