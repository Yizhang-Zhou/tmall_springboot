[来源](https://github.com/songxinjianqwe/Chat)
## 设计功能
1. 当前登陆的用户可以查看当前所有登陆的用户列表。然后选择其中用户发送消息。接收者不存在或已下线时将无法发送，并提示：接收者不存在或已下线
2. 用户收到其他人发送的消息时会提醒。因此这里有一个监听。

## 服务端
1. 维护所有登陆用户信息：登陆下线，服务端需要处理。用户已上线时这种通知也需要。
2. MessageHandler抽象类，里面有个重要方法broadcast。
3. 服务端代码参考ChatServer.java

## 客户端
1. 他直接来一个客户端继承了frame，我这里就是前端网页吧。然后登陆时就是连接服务器。

## 问题：
1. 还不清楚他的HTTP这个包的用途，task包应该核心？
2. 后期有空增加聊天时发送图片/文件的功能。

## 编码过程：
1. 写服务端。这个服务端工程新建，不与原先工程在一起。客户端就在原先的工程中吧。
   - 服务器需要监听是否有客户端连接。即ListenerThread
     - 然后需要处理客户端的连接请求，handleAcceptRequest
     - 需要处理客户端发送的数据。即ReadEventHandler。客户端发送的消息分下线通知、发送给其他人的消息等，因此需要分开处理。均继承ChatMessageHandler。到底是normal还是提示，分别处理。利用@Component("normal")把普通pojo实例化到spring容器中，然后通过类名访问applicationcontext.getBean("normal")。
     
2. 客户端。这个工程新建。
   - 初始化客户端的socket数据。
     - 登录，作为浏览器登录的附属品。除了网站的登录cookie，也作为服务器的登录附加。
     - 退出
   - 发送数据。这里为了数据统一管理，将发送服务器的数据作为message，而消息又分为消息体跟消息头。
     - 这里源码用的lombok的注解Builder，我选择直接使用链式建造者模式来编码。
   - 接受数据。同样为了管理数据，将接受的数据作为ChatResponse，即来自服务器的回应。
3. 依赖（没处理好，主工程即原先的tmall）
   - 常见工具放入common工程
   - server工程依赖主工程的user
   - server跟主工程都依赖common
   
4. 开始通信
   - 首先启动ChatServer、ChatClient。
   - 用户AB上线，发送上线消息给服务器，然后服务器解析消息后，用LoginMessageHandler将用户注册到UserManager。注册成功后服务器返回消息，服务器的消息统一用PromptMsgProperty来表示消息体。
   - 用户A向用户B发送消息，消息先发送到服务器（流的方式），服务器解析消息（先将流转字节数组，再重新生成消息对象），获取接受对象，服务器向接受对象发送消息（服务器获取用户与接收方的通道channel，然后向目标通道write）。这里就需要服务器来管理用户UserManager，然后向在线用户发消息。
   - 下线同理。
   
5. 重点：tmall与聊天项目的结合。
   - 通过httpclient获取tmall项目的用户数据，获取时为字符串，因此需要转换——JsonUtils工具包。`List<User> list = JsonUtils.jsonToList(json,User.class);`这样就获取所有用户信息。
   
   
5. 异常
   - 反射异常 java.lang.InstantiationException处理。在ChatServer中的`ChatMessage message = ProtoStuffUtil.deserialize(bytes, ChatMessage.class);`中反射生成对象失败。解决：使用反射的时候编写使用class实例化其他类的对象的时候，一定要自己定义无参的构造方法
   
6. 查看端口占用`netstat -aon|findstr "9000"`