#Flink on Yarn Kerberos 安全认证

Flink任务是如何提交到带有Kerberos认证的Yarn集群：
##为什么需要Kerberos
1. 在Hadoop1.0.0或者CDH3以前，Hadoop集群中的所有节点几乎就是裸奔的，主要存在以下安全问题：
2. NameNode与JobTracker上没有用户认证，用户可以伪装成管理员入侵到一个HDFS 或者MapReduce集群上。
3. DataNode上没有认证：Datanode对读入输出并没有认证，如果知道block的ID，就可以任意的访问DataNode上block的数据
4. JobTracker上没有认证：可以任意的杀死或更改用户的jobs，也可以更改JobTracker的工作状态
5. 没有对DataNode与TaskTracker的认证：用户可以伪装成DataNode与TaskTracker，去接受JobTracker与Namenode的任务指派。
为了解决这些问题，kerberos认证出现了，它实现的是机器级别的安全认证。

其原理是事先将集群中的机器添加到kerberos数据库中，在数据库中分别产生主机与各个节点的keytab，并将这些keytab分发到对应的节点上。通过这些keytab文件，节点可以从数据库中获得与目标节点通信的密钥防止身份被冒充。针对Hadoop集群可以解决两方面的认证:
1. 解决服务器到服务器的认证，确保不会冒充服务器的情况。集群中的机器都是是可靠的，有效防止了用户伪装成Datanode，Tasktracker，去接受JobTracker，Namenode的任务指派。
2. 解决client到服务器的认证，防止用户恶意冒充client提交作业，也无法发送对于作业的操作到JobTracker上，即使知道datanode的相关信息，也无法读取HDFS上的数据。
对于具体到用户粒度上的权限控制，如哪些用户可以提交某种类型的作业，哪些用户不能，目前Kerberos还没有实现，需要有专门的ACL模块进行把控。

##Kerberos认证过程
验证过程有三个参与者：Client、Server 和 KDC(Key Distribution Center);
其中KDC又分为两个部分：Authentication Service和Ticket Granting Service;
基本概念：
- Principal（安全个体）：被认证的个体，有一个名字和口令，每个server都对应一个principal，其格式如下，@前面部分为具体身份，后面的部分称为REALM。
component1 / component2 @ REALM
- KDC(Key Distribution Server)：是一个网络服务，提供ticket 和临时会话密钥；
- Ticket：一个记录，客户用来向服务器证明自己的身份，包括客户标识、会话密钥和时间戳；
- AS(Authentication Server): 认证服务器，隶属于KDC，用于认证Client身份；
- TGS(Ticket Granting Server): 许可证服务器，隶属于KDC。
事先对集群中确定的机器由管理员手动添加到kerberos数据库中，在KDC上分别产生主机与各个节点的keytab(包含了host和对应节点的名字，还有他们之间的密钥——Master Key)，并将这些keytab分发到对应的节点上。

![kerberos_exchange]()

###AS Exchange
通过这个过程，KDC（确切地说是KDC中的Authentication Service）可以实现对Client身份的确认，并颁发给该Client一个TGT(Ticket Granting Ticket)。
![as_exchange]()

1. Client向KDC的Authentication Service发送Authentication Service Request（KRB_AS_REQ）, 为了确保KRB_AS_REQ仅限于自己和KDC知道，Client使用自己的Master Key对KRB_AS_REQ的内容进行加密。KRB_AS_REQ内容主要包括：
- Pre-authentication data：用于证明自己知道自己声称的那个account的Password，一般是一个被Client的Master key加密过的Timestamp;
- Client信息: 可以理解为client的principal;
- TGS的Server Name。

2. AS在接收到的KRB_AS_REQ后从Account Database中提取Client对应的Master Key对Pre-authentication data进行解密，如果是一个合法的Timestamp，则可以证明发送方的确是Client信息中声称的那个人。
验证通过之后，AS将一份Authentication Service Response（KRB_AS_REP）发送给请求方。KRB_AS_REP主要包含两个部分：该Client的Master Key加密过的Session Key（SKDC-Client：Logon Session Key）和被自己（KDC）的Master Key加密的TGT。而TGT大体又包含以下的内容：
- Session Key；
- Client信息：即Client的principal；
- End time: TGT到期的时间。
Client通过自己的Master Key对第一部分解密获得Session Key之后，利用TGT便可以进行Kerberos认证的下一步：TGS Exchange。

###TGS Exchange
Client先向TGS发送Ticket Granting Service Request（KRB_TGS_REQ），其主要内容为：
- 被KDC的Master Key加密的TGT；
- Authenticator：用于验证确认Client提供的那个TGT是否是AS颁发给它的，其内容为Client信息与Timestamp，并且用Session Key进行加密。
- Client信息；
- Server信息：Client试图访问的Server的Principal。

1. TGS收到KRB_TGS_REQ后，先使用他自己的Master Key对TGT进行解密，从而获得Session Key。随后使用该Session Key解密Authenticator，通过比较Authenticator中的Client Info和Session Ticket中的Client Info从而实现对Client的验证。

![tgs_exchange]()

验证通过向对方发送Ticket Granting Service Response（KRB_TGS_REP）。这个KRB_TGS_REP有两部分组成：使用Logon Session Key（SKDC-Client）加密过用于Client和Server认证的Session Key（SServer-Client）和使用Server的Master Key进行加密的Ticket。该Ticket大体包含以下一些内容：
- Session Key：SServer-Client；
- Client信息；
- End time: Ticket的到期时间。

2. Client收到KRB_TGS_REP，使用Logon Session Key（SKDC-Client）解密第一部分后获得Session Key（SServer-Client）。有了Session Key和Ticket，Client就可以和Server进行交互，这时无须KDC介入了。我们看看 Client是如何使用Ticket与Server怎样进行交互的。

###CS(Client/Server) Exchange

![cs_exchange]()

1. 先是Client向Server认证自己的身份。这个过程与TGS Exchange中认证的过程类似，Client创建用于证明自己就是Ticket的真正所有者的Authenticator，并使用上一步获得的Session Key（SServer-Client）进行加密，然后将它和Ticket一起作为Application Service Request（KRB_AP_REQ）发送给Server。
除了上述两项内容之外，KRB_AP_REQ还包含一个Flag用于表示Client是否需要进行双向验证（Mutual Authentication）。

2. Server接收到KRB_AP_REQ之后，通过自己的Master Key解密Ticket，从而获得Session Key（SServer-Client）。通过Session Key（SServer-Client）解密Authenticator，进而验证对方的身份。验证成功，让Client访问需要访问的资源，否则直接拒绝对方的请求。
对于需要进行双向验证，Server从Authenticator提取Timestamp，使用Session Key（SServer-Client）进行加密，并将其发送给Client用于Client验证Server的身份。

##Flink on Kerberos Yarn实现方式
在客户端使用Kerberos认证来获取服务时，需要经过三个步骤:
1. 认证：客户端向认证服务器发送一条报文，并获取一个含时间戳的TGT。
2. 授权：客户端使用TGT向TGS请求一个服务Ticket。
3. 服务请求：客户端向服务器出示服务Ticket，以证实自己的合法性。
其关键在于获取TGT，客户端有了它就可以申请访问服务。

Flink链接Kafka只需配置Flink flink-conf.yaml即可为Kafka启用Kerberos身份验证，如下所示：
1. 通过设置以下内容配置Kerberos凭据：
security.kerberos.login.use-ticket-cache：默认为true并且Flink将尝试在所管理的票证缓存中使用Kerberos凭据kinit。请注意，在YARN上部署的Flink作业中使用Kafka连接器时，使用票证缓存的Kerberos授权将不起作用。使用Mesos进行部署时也是如此，因为Mesos部署不支持使用票证缓存进行授权。
security.kerberos.login.keytab和security.kerberos.login.principal：要改为使用Kerberos键表，请为这两个属性设置值。
2.附加KafkaClient到security.kerberos.login.contexts：告诉Flink向Kafka登录上下文提供配置的Kerberos凭据，以用于Kafka身份验证。
一旦启用了基于Kerberos的Flink安全性，就可以通过Flink Kafka消费者或生产者向Kafka进行身份验证，只需在传递给内部Kafka客户端的提供的属性配置中简单地包含以下两个设置即可：
- 设置security.protocol为SASL_PLAINTEXT【默认NONE】：用于与Kafka代理进行通信的协议。使用独立的Flink部署时，也可以使用SASL_SSL;
- 设置sasl.kerberos.service.name为kafka【默认kafka】：此值应与sasl.kerberos.service.name用于Kafka代理配置的值匹配。客户端和服务器配置之间的服务名称不匹配将导致身份验证失败。
