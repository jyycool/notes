# 【Kerberos】认证原理

Kerberos 在古希腊神话中是指：一只有三个头的狗。这条狗守护在地狱之门外，防止活人闯入。Kerberos 协议以此命名，因为协议的重要组成部分也是三个：client, server, KDC(密钥分发中心)。要了解 Kerberos 协议的工作过程，先了解不含 KDC 的简单相互身份验证过程。

## 一、 基本原理

Authentication 解决的是“如何证明某个人确确实实就是他或她所声称的那个人”的问题。

我们都耳熟能详小白兔和大灰狼的故事, 大灰狼冒充兔妈妈要小白兔开门, 这个过程就是 Kerberos 的认证过程: 我们假设小白兔和兔妈妈之间有一个验证身份的秘密(secret), 那么当大灰狼冒充兔妈妈要小白兔开门时, 小白兔就可以要求大灰狼提供这个秘密(secret)来证明它是兔妈妈。

这个过程实际上涉及到3个重要的关于 Authentication(认证) 的方面：

- Secret 如何表示。
- 大灰狼如何向小白兔提供 Secret。
- 小白兔如何识别 Secret。

基于这 3 个方面，我们把 Kerberos Authentication 进行最大限度的简化：整个过程涉及到Client 和 Server，他们之间的这个 Secret 我们用一个Key（**KServer-Client**）来表示。Client 为了让 Server 对自己进行有效的认证，向对方提供如下两组信息：

- 代表 Client 自身 Identity 的信息，为了简便，它以明文的形式传递。
- 将 Client 的 Identity 使用 **KServer-Client** 作为 Public Key、并采用对称加密算法进行加密。

由于 **KServer-Client** 仅仅被 Client 和 Server 知晓，所以被 Client 使用 KServer-Client 加密过的 Client Identity 只能被 Client 和 Server 解密。同理，Server 接收到 Client 传送的这两组信息，先通过 **KServer-Client** 对后者进行解密，随后将机密的数据同前者进行比较，如果完全一样，则可以证明 Client 能过提供正确的 **KServer-Client**，而这个世界上，仅仅只有真正的 Client 和自己知道 **KServer-Client**，所以可以对方就是他所声称的那个人。

![](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzAxLmpwZw?x-oss-process=image/format,png)

Keberos 大体上就是按照这样的一个原理来进行 Authentication 的。但是 Kerberos 远比这个复杂，在后续的章节中不断地扩充这个过程，知道 Kerberos 真实的认证过程。为了使读者更加容易理解后续的部分，在这里我们先给出两个重要的概念：

- **Long-term Key/Master Key**

  在 Security 的领域中，有的 Key 可能长期内保持不变，比如你在密码，可能几年都不曾改变，这样的 Key、以及由此派生的 Key 被称为Long-term Key。对于 Long-term Key 的使用有这样的原则：被 Long-term Key 加密的数据不应该在网络上传输。原因很简单，一旦这些被 Long-term Key 加密的数据包被恶意的网络监听者截获，在原则上，只要有充足的时间，他是可以通过计算获得你用于加密的 Long-term Key 的--任何加密算法都不可能做到绝对保密。

  在一般情况下，对于一个 Account 来说，密码往往仅仅限于该 Account 的所有者知晓，甚至对于任何 Domain 的 Administrator，密码仍然应该是保密的。但是密码却又是证明身份的凭据，所以必须通过基于你密码的派生的信息来证明用户的真实身份，在这种情况下，一般将你的密码进行 Hash 运算得到一个 Hash code, 我们一般管这样的 Hash Code 叫做 Master Key。由于 Hash Algorithm 是不可逆的，同时保证密码和 Master Key 是一一对应的，这样既保证了你密码的保密性，又同时保证你的 Master Key 和密码本身在证明你身份的时候具有相同的效力。

- **Short-term Key/Session Key**

  由于被 Long-term Key 加密的数据包不能用于网络传送，所以我们使用另一种 Short-term Key 来加密需要进行网络传输的数据。由于这种 Key 只在一段时间内有效，即使被加密的数据包被黑客截获，等他把 Key 计算出来的时候，这个 Key 早就已经过期了。

## 二、Key Distribution: KServer-Client从何而来

上面我们讨论了 Kerberos Authentication 的基本原理：通过让被认证的一方提供一个仅限于他和认证方知晓的 Key 来鉴定对方的真实身份。而被这个 Key 加密的数据包需要在 Client 和Server 之间传送，所以这个 Key 不能是一个 **Long-term Key**，而只可能是**Short-term Key**，这个 key 仅仅在 Client 和 Server 的一个 Session 中有效，所以我们称这个 Key 为Client 和 Server 之间的 Session Key（**SServer-Client**）。

现在我们来讨论 Client 和 Server 如何得到这个**SServer-Client**。在这里我们要引入一个重要的角色：**Kerberos Distribution Center-KDC**。KDC 在整个 Kerberos Authentication中作为 Client 和 Server 共同信任的第三方起着重要的作用，而 Kerberos 的认证过程就是通过这 3 方协作完成。

对于一个 Windows Domain 来说，**Domain Controller**扮演着 KDC 的角色。KDC 维护着一个存储着该 Domain 中所有帐户的 **Account Database**（一般地，这个 Account Database 由**AD** 来维护），也就是说，他知道属于每个 Account 的名称和派生于该 Account Password 的**Master Key**。而用于 Client 和 Server 相互认证的 **SServer-Client** 就是由 KDC 分发。下面我们来看看 KDC 分发 **SServer-Client** 的过程。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzAyLmpwZw?x-oss-process=image/format,png)

通过上图我们可以看到 KDC 分发 SServer-Client 的简单的过程：首先 Client 向 KDC 发送一个对 SServer-Client 的申请。这个申请的内容可以简单概括为“**我是某个 Client，我需要一个 Session Key 用于访问某个 Server** ”。KDC 在接收到这个请求的时候，生成一个 Session Key，为了保证这个 Session Key 仅仅限于发送请求的 Client和他希望访问的 Server 知晓，KDC 会为这个 Session Key 生成两个 Copy，分别被 Client 和Server 使用。然后从 Account database 中提取 Client 和 Server 的 Master Key 分别对这两个 Copy 进行对称加密。对于后者，和 Session Key 一起被加密的还包含关于 Client的一些信息。

KDC 现在有了两个分别被 Client 和 Server 的 Master Key 加密过的 Session Key，这两个 Session Key 如何分别被 Client 和 Server 获得呢？也许你马上会说，KDC 直接将这两个加密过的包发送给Client和Server不就可以了吗，但是如果这样做，对于Server来说会出现下面 两个问题：

- 由于一个 Server 会面对若干不同的 Client, 而每个 Client 都具有一个不同的 Session Key。那么 Server 就会为所有的 Client 维护这样一个 Session Key 的列表，这样做对于 Server 来说是比较麻烦而低效的。
- 由于网络传输的不确定性，可能出现这样一种情况：Client 很快获得 Session Key，并将这个 Session Key 作为 Credential 随同访问请求发送到 Server，但是用于 Server 的Session Key 确还没有收到，并且很有可能承载这个 Session Key 的永远也到不了 Server 端，Client 将永远得不到认证。

为了解决这个问题，Kerberos 的做法很简单，将这两个被加密的 Copy 一并发送给 Client，属于 Server 的那份由 Client 发送给Server。


可能有人会问，KDC 并没有真正去认证这个发送请求的 Client 是否真的就是那个他所声称的那个人，就把 Session Key 发送给他，会不会有什么问题？例如大灰狼声称自己是兔妈妈，它同样会得到兔妈妈和兔宝宝的 Session Key，这会不会有什么问题？实际上不存在问题，因为大灰狼声称自己是兔妈妈，KDC 就会使用兔妈妈的 Password 派生的 Master Key 对 Session Key 进行加密，所以只有真正知道兔妈妈的 Password 的才会通过解密获得 Session Key。 

## 三、Authenticator-为有效的证明自己提供证据

通过上面的过程，Client 实际上获得了两组信息：一个通过自己 Master Key 加密的 Session Key，另一个被 Server 的 Master Key 加密的数据包，包含 Session Key 和关于自己的一些确认信息。通过第一节，我们说只要通过一个双方知晓的 Key 就可以对对方进行有效的认证，但是在一个网络的环境中，这种简单的做法是具有安全漏洞。为此, Client 需要提供更多的证明信息，我们把这种证明信息称为 **Authenticator**，在 Kerberos 的 Authenticator 实际上就是**关于Client的一些信息** 和当前时间的一个 **Timestamp**（关于这个安全漏洞和Timestamp的作用，我将在后面解释）。

在这个基础上，我们再来看看 Server 如何对 Client 进行认证：Client 通过**Master Key** 对 KDC 加密的 Session Key 进行解密从而获得 **Session Key**，随后创建**Authenticator（Client Info + Timestamp）** 并用 **Session Key** 对其加密。最后连同从KDC 获得的、被 **Server的Master Key** 加密过的数据包（**Client Info + Session Key**）一并发送到 Server 端。我们把通过 Server 的 Master Key 加密过的数据包称为**Session Ticket**。

当 Server 接收到这两组数据后，先使用他 **自己的Master Key** 对 Session Ticket 进行解密，从而获得 **Session Key**。随后使用该 **Session Key** 解密 **Authenticator**，通过比较**Authenticator 中的 Client Info** 和 **Session Ticket 中的 Client Info**从而实现对Client的认证。

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzAzLmpwZw?x-oss-process=image/format,png)

### 为什么要使用 Timestamp ？

到这里，很多人可能认为这样的认证过程天衣无缝：只有当 Client 提供正确的 Session Key 方能得到 Server 的认证。但是在现实环境中，这存在很大的安全漏洞。

我们试想这样的现象：Client 向 Server 发送的数据包被某个恶意网络监听者截获，该监听者随后将数据包作为自己的 Credential 冒充该 Client 对 Server 进行访问，在这种情况下，依然可以很顺利地获得 Server 的成功认证。为了解决这个问题，Client 在 **Authenticator** 中会加入一个当前时间的 **Timestamp**。

在 Server 对 Authenticator 中的 Client Info 和 Session Ticket 中的 Client Info 进行比较之前，会先提取 Authenticator 中的 **Timestamp**，并同 **当前的时间** 进行比较，如果他们之间的偏差超出一个可以 **接受的时间范围（一般是5mins）**, Server会直接拒绝该Client 的请求。在这里需要知道的是，Server 维护着一个列表，这个列表记录着在这个可接受的时间范围内所有进行认证的 Client 和认证的时间。对于时间偏差在这个可接受的范围中的Client，Server 会从这个这个列表中获得 **最近一个该 Client 的认证时间**，只有当**Authenticator 中的 Timestamp 晚于通过一个 Client的最近的认证时间**的情况下，Server采用进行后续的认证流程。

### Time Synchronization 的重要性

上述基于 Timestamp 的认证机制只有在 Client 和 Server 端的时间保持同步的情况才有意义。所以保持 Time Synchronization 在整个认证过程中显得尤为重要。在一个 Domain 中，一般通过访问同一个 **Time Service** 获得当前时间的方式来实现时间的同步。

### 双向认证(Mutual Authentication)

Kerberos一个重要的优势在于它能够提供双向认证: 不但 Server 可以对 Client 进行认证，Client 也能对 Server 进行认证。

具体过程是这样的，如果 Client 需要对他访问的 Server 进行认证，会在它向 Server 发送的Credential 中设置一个是否需要认证的 Flag。Server 在对 Client 认证成功之后，会把Authenticator 中的 Timestamp 提出出来，通过 Session Key 进行加密，当 Client 接收到并使用 Session Key 进行解密之后，如果确认 **Timestamp** 和原来的完全一致，那么他可以认定 Server 的确是它试图访问的 Server。

那么为什么 Server 不直接把通过 Session Key 进行加密的 Authenticator 原样发送给Client，而要把 Timestamp 提取出来加密发送给 Client 呢？原因在于防止恶意的监听者通过获取的 Client 发送的 Authenticator 冒充 Server 获得 Client 的认证。

## 四、Ticket Granting Service

通过上面的介绍，我们发现 Kerberos 实际上一个基于 **Ticket** 的认证方式。Client 想要获取Server 端的资源，先得通过 Server 的认证；而认证的先决条件是 Client 向 Server 提供从KDC 获得的一个有 **Server 的 Master Key** 进行加密的**Session Ticket（Session Key + Client Info）**。可以这么说，Session Ticket 是 Client 进入 Server 领域的一张门票。而这张门票必须从一个合法的 Ticket 颁发机构获得，这个颁发机构就是 **Client 和 Server 双方信任的 KDC**， 同时这张 Ticket 具有超强的防伪标识：它是被 Server 的 Master Key 加密的。对 Client 来说，获得 Session Ticket 是整个认证过程中最为关键的部分。

上面我们只是简单地从大体上说明了 KDC 向 Client 分发 Ticket 的过程，而真正在 Kerberos 中的 Ticket Distribution 要复杂一些。为了更好的说明整个 Ticket Distribution 的过程，我在这里做一个类比。现在的股事很火爆，上海基本上是全民炒股，我就举一个认股权证的例子。有的上市公司在股票配股、增发、基金扩募、股份减持等情况会向公众发行**认股权证**，认股权证的持有人可以凭借这个权证认购一定数量的该公司股票，认股权证是一种具有看涨期权的金融衍生产品。

而我们今天所讲的 Client 获得 Ticket 的过程也和通过认股权证购买股票的过程类似。如果我们把 Client 提供给 Server 进行认证的 Ticket 比作股票的话，那么 Client 在从 KDC 那边获得 Ticket 之前，需要先获得这个 Ticket 的认购权证，这个认购权证在 Kerberos 中被称为**TGT：Ticket Granting Ticket**，TGT 的分发方仍然是 KDC。

我们现在来看看 Client 是如何从 KDC 处获得 TGT 的：首先 Client 向 KDC 发起对 TGT 的申请，申请的内容大致可以这样表示：“**我需要一张 TGT 用以申请获取用以访问所有 Server 的 Ticket**”。KDC 在收到该申请请求后，生成一个用于该 Client 和 KDC 进行安全通信的 **Session Key（SKDC-Client）**。为了保证该Session Key 仅供该 Client 和自己使用，KDC 使用 **Client 的 Master Key**和**自己的 Master Key** 对生成的 Session Key 进行加密，从而获得两个加密的 **SKDC-Client**的 Copy。对于后者，随 **SKDC-Client** 一起被加密的还包含以后用于鉴定 Client 身份的关于Client的一些信息。最后 KDC 将这两份 Copy 一并发送给 Client。这里有一点需要注意的是：为了免去 KDC 对于基于不同 Client 的 Session Key 进行维护的麻烦，就像 Server 不会保存 **Session Key（SServer-Client）**一样，KDC 也不会去保存这个 Session Key（**SKDC-Client**），而选择完全靠 Client 自己提供的方式。

 ![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzA3LmdpZg)
当 Client 收到 KDC 的两个加密数据包之后，先使用 **自己的Master Key** 对第一个 Copy 进行解密，从而获得 KDC 和 Client 的 **Session Key（SKDC-Client）**，并把该 Session 和TGT 进行缓存。有了 Session Key 和 TGT，Client 自己的 Master Key 将不再需要，因为此后 Client 可以使用 **SKDC-Client** 向 KDC 申请用以访问每个 Server 的 Ticket，相对于 Client 的 Master Key 这个 Long-term Key，SKDC-Client是一个Short-term Key，安全保证得到更好的保障，这也是 Kerberos 多了这一步的关键所在。同时需要注意的是 SKDC-Client 是一个 Session Key，他具有自己的生命周期，同时 TGT 和 Session 相互关联，当Session Key 过期，TGT 也就宣告失效，此后 Client 不得不重新向 KDC 申请新的TGT，KDC将会生成一个不同 Session Key 和与之关联的 TGT。同时，由于 Client Log off 也导致SKDC-Client 的失效，所以 SKDC-Client 又被称为 **Logon Session Key**。

接下来，我们看看 Client 如何使用 TGT 来从 KDC 获得基于某个 Server 的 Ticket。在这里我要强调一下，Ticket 是基于某个具体的 Server 的，而 TGT 则是和具体的 Server 无关的，Client 可以使用一个 TGT 从 KDC 获得基于不同 Server 的T icket。我们言归正传，Client 在获得自己和 KDC 的 **Session Key（SKDC-Client）** 之后，生成自己的Authenticator 以及所要访问的 Server 名称并使用 **SKDC-Client** 进行加密。随后连同 TGT 一并发送给 KDC。KDC 使用**自己的Master Key**对 TGT 进行解密，提取 Client Info 和**Session Key（SKDC-Client）**，然后使用这个 **SKDC-Client** 解密 Authenticator 获得Client Info，对两个 Client Info 进行比较进而验证对方的真实身份。验证成功，生成一份基于 Client 所要访问的 Server 的 Ticket 给 Client，这个过程就是我们第二节中介绍的一样了。 

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzA1LmdpZg)

## 五、3 个 Sub-protocol：整个Authentication

通过以上的介绍，我们基本上了解了整个 Kerberos authentication 的整个流程：整个流程大体上包含以下3个子过程：

1. Client 向 KDC 申请TGT（Ticket Granting Ticket）。
2. Client 通过获得 TGT 向 DKC 申请用于访问 Server 的 Ticket。
3. Client 最终向 Server 提交 Ticket 以提供自己的认证。

不过上面的介绍离真正的 Kerberos Authentication 还是有一点出入。Kerberos 整个认证过程通过 3 个sub-protocol 来完成。这 3 个 Sub-Protocol 分别完成上面列出的 3 个子过程。这 3 个 sub-protocol 分别为：

1. Authentication Service Exchange
2. Ticket Granting Service Exchange
3. Client/Server Exchange

下图简单展示了完成这 3 个 Sub-protocol 所进行 Message Exchange。

 ![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAxXzA2LmdpZg)

### 5.1 Authentication Service Exchange

通过这个 Sub-protocol，KDC（确切地说是 KDC 中的 Authentication Service）实现对 Client 身份的确认，并颁发给该 Client 一个 TGT。具体过程如下：

1. Client 向 KDC 的 Authentication Service 发送 Authentication Service Request（**KRB_AS_REQ**）, 为了确保 KRB_AS_REQ 仅限于自己和 KDC 知道，Client 使用自己的 Master Key 对 KRB_AS_REQ 的主体部分进行加密（KDC可以通过 Domain 的 Account Database 获得该 Client 的 Master Key）。KRB_AS_REQ的大体包含以下的内容：

   - Pre-authentication data

     包含用以证明自己身份的信息。说白了，就是证明自己知道自己声称的那个account的Password。一般地，它的内容是一个被Client的Master key加密过的Timestamp。

   - Client name & realm

     简单地说就是Domain name\Client

   - Server Name

     注意这里的 Server Name 并不是 Client 真正要访问的 Server 的名称，而我们也说了 TGT 是和 Server 无关的（Client 只能使用 Ticket，而不是 TGT 去访问 Server）。这里的 Server Name 实际上是**KDC 的 Ticket Granting Service 的 Server Name**。

2. AS（Authentication Service）通过它接收到的 KRB_AS_REQ 验证发送方的是否是在 Client name & realm 中声称的那个人，也就是说要验证发送放是否知道 Client的 Password。所以 AS 只需从 Account Database 中提取 Client 对应的 Master Key 对 Pre-authentication data 进行解密，如果是一个合法的Timestamp，则可以证明发送放提供的是正确无误的密码。验证通过之后，AS 将一份Authentication Service Response（KRB_AS_REP）发送给 Client。

   KRB_AS_REQ 主要包含两个部分：本 Client 的 Master Key 加密过的 Session Key（SKDC-Client：Logon Session Key）和被自己（KDC）加密的TGT。而TGT大体又包含以下的内容：

   - Session Key: SKDC-Client：Logon Session Key

   - Client name & realm: 简单地说就是Domain name\Client

   - End time: TGT到期的时间。

3. Client 通过自己的 Master Key 对第一部分解密获得 Session Key（SKDC-Client：Logon Session Key）之后，携带着 TGT 便可以进入下一步：TGS（Ticket Granting Service）Exchange。

### 5.2 TGS(Ticket Granting Service) Exchange

1. TGS（Ticket Granting Service）Exchange 通过 Client 向 KDC 中的 TGS（Ticket Granting Service）发送 Ticket Granting Service Request（**KRB_TGS_REQ**）开始。KRB_TGS_REQ 大体包含以下的内容：

   - TGT - Client 通过 AS Exchange 获得的 Ticket Granting Ticket，TGT 被 KDC 的 Master Key 进行加密。

   - Authenticator - 用以证明当初 TGT 的拥有者是否就是自己，所以它必须以 TGT 的办法方和自己的Session Key（SKDC-Client：Logon Session Key）来进行加密。

   - Client name & realm - 简单地说就是Domain name\Client。

   - Server name & realm - 简单地说就是Domain name\Server，这回是Client试图访问的那个Server。

2. TGS 收到 KRB_TGS_REQ 在发给 Client 真正的 Ticket 之前，先得整个 Client提供的那个 TGT 是否是 AS 颁发给它的。于是它不得不通过 Client 提供的Authenticator 来证明。但是 Authentication 是通过**Logon Session Key（SKDC-Client）** 进行加密的，而自己并没有保存这个 Session Key。所以 TGS 先得通过自己的 Master Key 对 Client 提供的 TGT 进行解密，从而获得这个 Logon Session Key（SKDC-Client），再通过这个**Logon Session Key（SKDC-Client）** 解密 Authenticator 进行验证。验证通过向对方发送 Ticket Granting Service Response（KRB_TGS_REP）。这个KRB_TGS_REP 有两部分组成：使用 **Logon Session Key（SKDC-Client）** 加密过用于 Client 和 Server 的 **Session Key（SServer-Client）** 和使用**Server的Master Key** 进行加密的 Ticket。该 Ticket大体包含以下一些内容：

   - Session Key - SServer-Client。

   - Client name & realm - 简单地说就是Domain name\Client。

   - End time - Ticket的到期时间。

3. Client 收到 KRB_TGS_REP，使用 **Logon Session Key（SKDC-Client）** 解密第一部分后获得 **Session Key（SServer-Client）**。有了 Session Key 和 Ticket，Client 就可以之间和 Server 进行交互，而无须在通过 KDC 作中间人了。所以我们说 Kerberos 是一种高效的认证方式，它可以直接通过 Client 和 Server双方来完成，不像 Windows NT 4 下的 NTLM 认证方式，每次认证都要通过一个双方信任的第 3 方来完成。

我们现在来看看 Client 如果使用 Ticket 和 Server 怎样进行交互的，这个阶段通过我们的第3个 Sub-protocol 来完成：**CS（Client/Server ）Exchange**。

### 5.3 CS(Client/Server) Exchange

这个已经在本文的第二节中已经介绍过，对于重复发内容就不再累赘了。Client 通过 TGS Exchange 获得 Client 和 Server 的 **Session Key（SServer-Client）**，随后创建用于证明自己就是 Ticket 的真正所有者的 Authenticator，并使用**Session Key（SServer-Client）** 进行加密。最后将这个被加密过的 Authenticator 和 Ticket 作为 Application Service Request（KRB_AP_REQ）发送给 Server。除了上述两项内容之外，KRB_AP_REQ 还包含一个 Flag 用于表示 Client 是否需要进行双向验证（Mutual Authentication）。

Server 接收到 KRB_AP_REQ 之后，通过自己的 Master Key 解密 Ticket，从而获得Session Key（SServer-Client）。通过 Session Key（SServer-Client）解密Authenticator，进而验证对方的身份。验证成功，让 Client 访问需要访问的资源，否则直接拒绝对方的请求。

对于需要进行双向验证，Server从Authenticator提取Timestamp，使用Session Key（SServer-Client）进行加密，并将其发送给Client用于Client验证Server的身份。



## 六、User2User Sub-Protocol：有效地保障Server的安全

通过 3 个 Sub-protocol 的介绍，我们可以全面地掌握整个 Kerberos 的认证过程。实际上，在 Windows 2000 时代，基于 Kerberos 的 Windows Authentication 就是按照这样的工作流程来进行的。但是我在上面一节结束的时候也说了，基于 3 个 Sub-protocol 的 Kerberos 作为一种 Network Authentication 是具有它自己的局限和安全隐患的。

我在整篇文章一直在强调这样的一个原则: `以某个 Entity 的 Long-term Key 加密的数据不应该在网络中传递。`原因很简单，所有的加密算法都不能保证100%的安全，对加密的数据进行解密只是一个时间的过程，最大限度地提供安全保障的做法就是: `使用一个Short-term key（Session Key）代替 Long-term Key 对数据进行加密，使得恶意用户对其解密获得加密的Key时，该 Key 早已失效。`

但是对于 3 个 Sub-Protocol 的 C/S Exchange，Client 携带的 Ticket 却是被**Server Master Key** 进行加密的，这显现不符合我们提出的原则，降低 Server 的安全系数。所以我们必须寻求一种解决方案来解决上面的问题。这个解决方案很明显：就是采用一个Short-term 的 Session Key，而不是 Server Master Key 对 Ticket 进行加密。

这就是我们今天要介绍的 Kerberos 的第 4 个 Sub-protocol：**User2User Protocol**。我们知道，既然是 Session Key，仅必然涉及到两方，而在 Kerberos 整个认证过程涉及到3方：Client、Server和KDC，所以用于加密 Ticket 的只可能是 Server 和 KDC 之间的 **Session Key（SKDC-Server）。**

我们知道 Client 通过在 AS Exchange 阶段获得的 TGT 从 KDC 那么获得访问 Server 的 Ticket。原来的 Ticket 是通过 **Server的Master Key** 进行加密的，而这个 Master Key 可以通过 Account Database 获得。但是现在 KDC 需要使用 Server和 KDC 之间的 **SKDC-Server ** 进行加密，`而 KDC 是不会维护这个 Session Key，所以这个 Session Key 只能靠申请 Ticket 的 Client提供。`所以在 AS Exchange 和TGS Exchange 之间，Client 还得对 Server 进行请求已获得 Server 和 KDC 之间的Session Key（**SKDC-Server**）。而对于 Server 来说，它可以像 Client 一样通过**AS Exchange** 获得他和 KDC 之间的 Session Key（**SKDC-Server**）和一个封装了这个Session Key 并被 **KDC的Master Key进行加密的TGT**，一旦获得这个 TGT，Server 会缓存它，以待 Client 对它的请求。我们现在来详细地讨论这一过程。

 

![img](https://imgconvert.csdnimg.cn/aHR0cDovL3d3dy5jbmJsb2dzLmNvbS9pbWFnZXMvY25ibG9nc19jb20vYXJ0ZWNoL2tlcmJlcm9zXzAzXzAxLmdpZg)
上图基本上翻译了基于 User2User 的认证过程，这个过程由 4 个步骤组成。我们发现较之我在上面一节介绍的基于传统 3 个 Sub-protocol 的认证过程，这次对了第2部。我们从头到尾简单地过一遍：

1. AS Exchange：Client 通过此过程获得了属于自己的 TGT，有了此 TGT，Client 可凭此向 KDC 申请用于访问某个 Server 的 Ticket。
2. 这一步的主要任务是获得封装了 Server 和 KDC 的 Session Key（SKDC-Server）的属于 Server 的 TGT。如果该 TGT 存在于 Server 的缓存中，则 Server 会直接将其返回给 Client。否则通过 AS Exchange 从 KDC 获取。
3. TGS Exchange：Client 通过向 KDC 提供自己的 TGT，Server 的 TGT 以及Authenticator 向 KDC 申请用于访问 Server 的 Ticket。KDC 使用先用自己的Master Key 解密 Client 的 TGT 获得 SKDC-Client，通过 SKDC-Client 解密Authenticator 验证发送者是否是 TGT 的真正拥有者，验证通过再用自己的 Master Key 解密 Server 的 TGT 获得 KDC 和 Server 的 Session Key（SKDC-Server），并用该 Session Key 加密 Ticket 返回给 Client。
4. C/S Exchange：Client 携带者通过 KDC 和 Server 的 Session Key（SKDC-Server）进行加密的 Ticket 和通过 Client 和 Server 的 Session Key（SServer-Client）的 Authenticator 访问 Server，Server 通过 SKDC-Server 解密 Ticket 获得 SServer-Client，通过 SServer-Client 解密Authenticator 实现对 Client 的验证。

这就是整个过程。

## 七、Kerberos的优点

分析整个 Kerberos 的认证过程之后，我们来总结一下 Kerberos 都有哪些优点：

1. 较高的 Performance

   虽然我们一再地说 Kerberos 是一个涉及到3方的认证过程：Client、Server、KDC。但是一旦 Client 获得用过访问某个 Server 的 Ticket，该 Server 就能根据这个Ticket 实现对 Client 的验证，而无须 KDC 的再次参与。和传统的基于 Windows NT 4.0 的每个完全依赖 Trusted Third Party 的 NTLM 比较，具有较大的性能提升。

2. 实现了双向验证 (Mutual Authentication)

   传统的 NTLM 认证基于这样一个前提：Client 访问的远程的 Service 是可信的、无需对于进行验证，所以 NTLM 不曾提供双向验证的功能。这显然有点理想主义，为此Kerberos 弥补了这个不足：Client 在访问 Server 的资源之前，可以要求对 Server 的身份执行认证。

3. 对 Delegation 的支持

   Impersonation 和 Delegation 是一个分布式环境中两个重要的功能。Impersonation 允许 Server 在本地使用 Logon 的 Account 执行某些操作，Delegation 需用 Server 将 logon 的 Account 带入到另过一个 Context 执行相应的操作。NTLM 仅对 Impersonation 提供支持，而 Kerberos通 过一种双向的、可传递的（Mutual 、Transitive）信任模式实现了对 Delegation 的支持。

4. 互操作性(Interoperability)

   Kerberos 最初由 MIT 首创，现在已经成为一行被广泛接受的标准。所以对于不同的平台可以进行广泛的互操作。