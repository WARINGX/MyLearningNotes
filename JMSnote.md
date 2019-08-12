### 什么是消息？

​		消息传递实现**松散耦合的**分布式通信。组件将消息发送到目标，并且收件人可以从目标检索消息。但是，发送者和接收者不必同时可用以进行通信。事实上，发送者不需要知道接收器的任何信息; 接收者也不需要了解发送者的任何信息。发送方和接收方只需知道使用哪种消息格式和目的地。在这方面，消息传递不同于紧密耦合的技术，例如远程方法调用（RMI），它需要应用程序知道远程应用程序的方法。消息传递也不同于电子邮件（电子邮件），电子邮件是人们之间或软件应用程序与人之间的通信方法。消息传递用于软件应用程序或软件组件之间的通信。



### 什么是JMS API？

​		Java消息服务是一种Java API，允许应用程序创建，发送，接收和读取消息。JMS API由Sun和几家合作伙伴公司设计，定义了一组通用的接口和相关语义，允许用Java编程语言编写的程序与其他消息传递实现进行通信。JMS API最大限度地减少了程序员为了使用消息传递产品而必须学习的概念集，但提供了足够的功能来支持复杂的消息传递应用程序。它还致力于在同一消息传递域中最大化JMS应用程序之间的JMS应用程序的可移植性。

JMS API使通信不仅松散耦合，而且：

- **异步**：JMS提供程序可以在客户端到达时将消息传递给客户端; 客户端不必请求消息以接收消息。
- **可靠**：JMS API可以确保消息只传送一次。较低级别的可靠性适用于能够错过消息或接收重复消息的应用程序。

### 什么时候可以使用JMS API？

在下列情况下，企业应用程序提供商可能会在紧密耦合的API上选择消息传递API，例如远程过程调用（RPC）。

- 提供商希望组件不依赖于其他组件接口的信息，因此可以轻松替换组件。
- 无论所有组件是否同时启动和运行，提供程序都希望应用程序运行。
- 应用程序业务模型允许组件将信息发送给另一个组件并继续运行而不会立即响应。

例如，汽车制造商的企业应用程序的组件可以在以下情况下使用JMS API：

- 当产品的库存水平低于某个水平时，库存组件可以向工厂组件发送消息，以便工厂可以生产更多的汽车。
- 工厂组件可以向零件组件发送消息，以便工厂可以组装所需的零件。
- 零件组件又可以将消息发送到他们自己的库存并订购组件以更新他们的库存并从供应商订购新零件。
- 工厂和零件组件都可以向会计组件发送消息以更新预算编号。
- 企业可以将更新的目录项发布给其销售人员。

对这些任务使用消息传递允许各种组件有效地相互交互，而不会占用网络或其他资源。下图说明了这个简单示例如何工作。

![该图显示了企业中各部门之间的消息传递](https://docs.oracle.com/javaee/6/tutorial/doc/figures/jms-msgEnterpriseApp.gif)

## *ActiveMQ详解

#### 一、JMS的两种消息模式

##### 允许发送消息的数据类型

```
            //纯字符串的数据
            session.createTextMessage();
            //序列化的对象
            session.createObjectMessage();
            //流，可以用来传递文件等
            session.createStreamMessage();
            //用来传递字节
            session.createBytesMessage();
            //这个方法创建出来的就是一个map，可以把它当作map来用，当你看了它的一些方法，你就懂了
            session.createMapMessage();
            //这个方法，拿到的是javax.jms.Message，是所有message的接口
            session.createMessage();
```

#### 1、点对点的消息模式

点对点的模式主要建立在一个队列上面，当连接一个列队的时候，发送端不需要知道接收端是否正在接收，可以直接向ActiveMQ发送消息，发送的消息，将会先进入队列中，如果有接收端在监听，则会发向接收端，如果没有接收端接收，则会保存在activemq服务器，直到接收端接收消息，点对点的消息模式可以有多个发送端，多个接收端，但是一条消息，只会被一个接收端给接收到，哪个接收端先连上ActiveMQ，则会先接收到，而后来的接收端则接收不到那条消息

这里使用java来实现一下ActiveMQ的点对点模式。

###### **项目使用MAVEN来构建**

```
    <dependencies>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-core</artifactId>
            <version>5.7.0</version>
        </dependency>
    </dependencies>
```

###### **点对点的发送端**

```
import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.DeliveryMode;
import javax.jms.Destination;
import javax.jms.JMSException;
import javax.jms.MessageProducer;
import javax.jms.Session;
import javax.jms.TextMessage;

import org.apache.activemq.ActiveMQConnectionFactory;

public class PTPSend {
    //连接账号
    private String userName = "";
    //连接密码
    private String password = "";
    //连接地址
    private String brokerURL = "tcp://192.168.1.123:1234";
    //connection的工厂
    private ConnectionFactory factory;
    //连接对象
    private Connection connection;
    //一个操作会话
    private Session session;
    //目的地，其实就是连接到哪个队列，如果是点对点，那么它的实现是Queue，如果是订阅模式，那它的实现是Topic
    private Destination destination;
    //生产者，就是产生数据的对象
    private MessageProducer producer;
    
    public static void main(String[] args) {
        PTPSend send = new PTPSend();
        send.start();
    }
    
    public void start(){
        try {
            //根据用户名，密码，url创建一个连接工厂
            factory = new ActiveMQConnectionFactory(userName, password, brokerURL);
            //从工厂中获取一个连接
            connection = factory.createConnection();
            //测试过这个步骤不写也是可以的，但是网上的各个文档都写了
            connection.start();
            //创建一个session
            //第一个参数:是否支持事务，如果为true，则会忽略第二个参数，被jms服务器设置为SESSION_TRANSACTED
            //第二个参数为false时，paramB的值可为Session.AUTO_ACKNOWLEDGE，Session.CLIENT_ACKNOWLEDGE，DUPS_OK_ACKNOWLEDGE其中一个。
            //Session.AUTO_ACKNOWLEDGE为自动确认，客户端发送和接收消息不需要做额外的工作。哪怕是接收端发生异常，也会被当作正常发送成功。
            //Session.CLIENT_ACKNOWLEDGE为客户端确认。客户端接收到消息后，必须调用javax.jms.Message的acknowledge方法。jms服务器才会当作发送成功，并删除消息。
            //DUPS_OK_ACKNOWLEDGE允许副本的确认模式。一旦接收方应用程序的方法调用从处理消息处返回，会话对象就会确认消息的接收；而且允许重复确认。
            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            //创建一个到达的目的地，其实想一下就知道了，activemq不可能同时只能跑一个队列吧，这里就是连接了一个名为"text-msg"的队列，这个会话将会到这个队列，当然，如果这个队列不存在，将会被创建
            destination = session.createQueue("text-msg");
            //从session中，获取一个消息生产者
            producer = session.createProducer(destination);
            //设置生产者的模式，有两种可选
            //DeliveryMode.PERSISTENT 当activemq关闭的时候，队列数据将会被保存
            //DeliveryMode.NON_PERSISTENT 当activemq关闭的时候，队列里面的数据将会被清空
            producer.setDeliveryMode(DeliveryMode.PERSISTENT);
            
            //创建一条消息，当然，消息的类型有很多，如文字，字节，对象等,可以通过session.create..方法来创建出来
            TextMessage textMsg = session.createTextMessage("呵呵");
            for(int i = 0 ; i < 100 ; i ++){
                //发送一条消息
                producer.send(textMsg);
            }
            
            System.out.println("发送消息成功");
            //即便生产者的对象关闭了，程序还在运行哦
            producer.close();
            
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```

###### **点对点的接收端**

```
import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.Destination;
import javax.jms.JMSException;
import javax.jms.Message;
import javax.jms.MessageConsumer;
import javax.jms.MessageListener;
import javax.jms.Session;
import javax.jms.TextMessage;

import org.apache.activemq.ActiveMQConnectionFactory;

public class PTPReceive {
    //连接账号
    private String userName = "";
    //连接密码
    private String password = "";
    //连接地址
    private String brokerURL = "tcp://192.168.1.123:1234";
    //connection的工厂
    private ConnectionFactory factory;
    //连接对象
    private Connection connection;
    //一个操作会话
    private Session session;
    //目的地，其实就是连接到哪个队列，如果是点对点，那么它的实现是Queue，如果是订阅模式，那它的实现是Topic
    private Destination destination;
    //消费者，就是接收数据的对象
    private MessageConsumer consumer;
    public static void main(String[] args) {
        PTPReceive receive = new PTPReceive();
        receive.start();
    }
    
    public void start(){
        try {
            //根据用户名，密码，url创建一个连接工厂
            factory = new ActiveMQConnectionFactory(userName, password, brokerURL);
            //从工厂中获取一个连接
            connection = factory.createConnection();
            //测试过这个步骤不写也是可以的，但是网上的各个文档都写了
            connection.start();
            //创建一个session
            //第一个参数:是否支持事务，如果为true，则会忽略第二个参数，被jms服务器设置为SESSION_TRANSACTED
            //第二个参数为false时，paramB的值可为Session.AUTO_ACKNOWLEDGE，Session.CLIENT_ACKNOWLEDGE，DUPS_OK_ACKNOWLEDGE其中一个。
            //Session.AUTO_ACKNOWLEDGE为自动确认，客户端发送和接收消息不需要做额外的工作。哪怕是接收端发生异常，也会被当作正常发送成功。
            //Session.CLIENT_ACKNOWLEDGE为客户端确认。客户端接收到消息后，必须调用javax.jms.Message的acknowledge方法。jms服务器才会当作发送成功，并删除消息。
            //DUPS_OK_ACKNOWLEDGE允许副本的确认模式。一旦接收方应用程序的方法调用从处理消息处返回，会话对象就会确认消息的接收；而且允许重复确认。
            session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            //创建一个到达的目的地，其实想一下就知道了，activemq不可能同时只能跑一个队列吧，这里就是连接了一个名为"text-msg"的队列，这个会话将会到这个队列，当然，如果这个队列不存在，将会被创建
            destination = session.createQueue("text-msg");
            //根据session，创建一个接收者对象
            consumer = session.createConsumer(destination);
            
            
            //实现一个消息的监听器
            //实现这个监听器后，以后只要有消息，就会通过这个监听器接收到
            consumer.setMessageListener(new MessageListener() {
                @Override
                public void onMessage(Message message) {
                    try {
                        //获取到接收的数据
                        String text = ((TextMessage)message).getText();
                        System.out.println(text);
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            });
            //关闭接收端，也不会终止程序哦
//            consumer.close();
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
```



#### 2、发布/订阅消息模式

订阅/发布模式，同样可以有着多个发送端与多个接收端，但是接收端与发送端存在时间上的依赖，就是如果发送端发送消息的时候，接收端并没有监听消息，那么ActiveMQ将不会保存消息，将会认为消息已经发送，换一种说法，就是发送端发送消息的时候，接收端不在线，是接收不到消息的，哪怕以后监听消息，同样也是接收不到的。这个模式还有一个特点，那就是，发送端发送的消息，将会被所有的接收端给接收到，不类似点对点，一条消息只会被一个接收端给接收到。

###### **订阅模式的发送端**

```
import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.DeliveryMode;
import javax.jms.Destination;
import javax.jms.JMSException;
import javax.jms.MessageProducer;
import javax.jms.Session;
import javax.jms.TextMessage;

import org.apache.activemq.ActiveMQConnectionFactory;

public class TOPSend {
    //连接账号
        private String userName = "";
        //连接密码
        private String password = "";
        //连接地址
        private String brokerURL = "tcp://192.168.0.130:61616";
        //connection的工厂
        private ConnectionFactory factory;
        //连接对象
        private Connection connection;
        //一个操作会话
        private Session session;
        //目的地，其实就是连接到哪个队列，如果是点对点，那么它的实现是Queue，如果是订阅模式，那它的实现是Topic
        private Destination destination;
        //生产者，就是产生数据的对象
        private MessageProducer producer;
        
        public static void main(String[] args) {
            TOPSend send = new TOPSend();
            send.start();
        }
        
        public void start(){
            try {
                //根据用户名，密码，url创建一个连接工厂
                factory = new ActiveMQConnectionFactory(userName, password, brokerURL);
                //从工厂中获取一个连接
                connection = factory.createConnection();
                //测试过这个步骤不写也是可以的，但是网上的各个文档都写了
                connection.start();
                //创建一个session
                //第一个参数:是否支持事务，如果为true，则会忽略第二个参数，被jms服务器设置为SESSION_TRANSACTED
                //第二个参数为false时，paramB的值可为Session.AUTO_ACKNOWLEDGE，Session.CLIENT_ACKNOWLEDGE，DUPS_OK_ACKNOWLEDGE其中一个。
                //Session.AUTO_ACKNOWLEDGE为自动确认，客户端发送和接收消息不需要做额外的工作。哪怕是接收端发生异常，也会被当作正常发送成功。
                //Session.CLIENT_ACKNOWLEDGE为客户端确认。客户端接收到消息后，必须调用javax.jms.Message的acknowledge方法。jms服务器才会当作发送成功，并删除消息。
                //DUPS_OK_ACKNOWLEDGE允许副本的确认模式。一旦接收方应用程序的方法调用从处理消息处返回，会话对象就会确认消息的接收；而且允许重复确认。
                session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
                //创建一个到达的目的地，其实想一下就知道了，activemq不可能同时只能跑一个队列吧，这里就是连接了一个名为"text-msg"的队列，这个会话将会到这个队列，当然，如果这个队列不存在，将会被创建
                
                
                
                //=======================================================
                //点对点与订阅模式唯一不同的地方，就是这一行代码，点对点创建的是Queue，而订阅模式创建的是Topic
                destination = session.createTopic("topic-text");
                //=======================================================
                
                
                
                
                //从session中，获取一个消息生产者
                producer = session.createProducer(destination);
                //设置生产者的模式，有两种可选
                //DeliveryMode.PERSISTENT 当activemq关闭的时候，队列数据将会被保存
                //DeliveryMode.NON_PERSISTENT 当activemq关闭的时候，队列里面的数据将会被清空
                producer.setDeliveryMode(DeliveryMode.PERSISTENT);
                
                //创建一条消息，当然，消息的类型有很多，如文字，字节，对象等,可以通过session.create..方法来创建出来
                TextMessage textMsg = session.createTextMessage("哈哈");
                long s = System.currentTimeMillis();
                for(int i = 0 ; i < 100 ; i ++){
                    //发送一条消息
                    producer.send(textMsg);
                }
                long e = System.currentTimeMillis();
                System.out.println("发送消息成功");
                System.out.println(e - s);
                //即便生产者的对象关闭了，程序还在运行哦
                producer.close();
                
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
}
```

###### **订阅模式的接收端**

```
import javax.jms.Connection;
import javax.jms.ConnectionFactory;
import javax.jms.DeliveryMode;
import javax.jms.Destination;
import javax.jms.JMSException;
import javax.jms.MessageProducer;
import javax.jms.Session;
import javax.jms.TextMessage;

import org.apache.activemq.ActiveMQConnectionFactory;

public class TOPSend {
    //连接账号
        private String userName = "";
        //连接密码
        private String password = "";
        //连接地址
        private String brokerURL = "tcp://192.168.0.130:61616";
        //connection的工厂
        private ConnectionFactory factory;
        //连接对象
        private Connection connection;
        //一个操作会话
        private Session session;
        //目的地，其实就是连接到哪个队列，如果是点对点，那么它的实现是Queue，如果是订阅模式，那它的实现是Topic
        private Destination destination;
        //生产者，就是产生数据的对象
        private MessageProducer producer;
        
        public static void main(String[] args) {
            TOPSend send = new TOPSend();
            send.start();
        }
        
        public void start(){
            try {
                //根据用户名，密码，url创建一个连接工厂
                factory = new ActiveMQConnectionFactory(userName, password, brokerURL);
                //从工厂中获取一个连接
                connection = factory.createConnection();
                //测试过这个步骤不写也是可以的，但是网上的各个文档都写了
                connection.start();
                //创建一个session
                //第一个参数:是否支持事务，如果为true，则会忽略第二个参数，被jms服务器设置为SESSION_TRANSACTED
                //第二个参数为false时，paramB的值可为Session.AUTO_ACKNOWLEDGE，Session.CLIENT_ACKNOWLEDGE，DUPS_OK_ACKNOWLEDGE其中一个。
                //Session.AUTO_ACKNOWLEDGE为自动确认，客户端发送和接收消息不需要做额外的工作。哪怕是接收端发生异常，也会被当作正常发送成功。
                //Session.CLIENT_ACKNOWLEDGE为客户端确认。客户端接收到消息后，必须调用javax.jms.Message的acknowledge方法。jms服务器才会当作发送成功，并删除消息。
                //DUPS_OK_ACKNOWLEDGE允许副本的确认模式。一旦接收方应用程序的方法调用从处理消息处返回，会话对象就会确认消息的接收；而且允许重复确认。
                session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
                //创建一个到达的目的地，其实想一下就知道了，activemq不可能同时只能跑一个队列吧，这里就是连接了一个名为"text-msg"的队列，这个会话将会到这个队列，当然，如果这个队列不存在，将会被创建
                
                
                
                //=======================================================
                //点对点与订阅模式唯一不同的地方，就是这一行代码，点对点创建的是Queue，而订阅模式创建的是Topic
                destination = session.createTopic("topic-text");
                //=======================================================
                
                
                
                
                //从session中，获取一个消息生产者
                producer = session.createProducer(destination);
                //设置生产者的模式，有两种可选
                //DeliveryMode.PERSISTENT 当activemq关闭的时候，队列数据将会被保存
                //DeliveryMode.NON_PERSISTENT 当activemq关闭的时候，队列里面的数据将会被清空
                producer.setDeliveryMode(DeliveryMode.PERSISTENT);
                
                //创建一条消息，当然，消息的类型有很多，如文字，字节，对象等,可以通过session.create..方法来创建出来
                TextMessage textMsg = session.createTextMessage("哈哈");
                long s = System.currentTimeMillis();
                for(int i = 0 ; i < 100 ; i ++){
                    //发送一条消息
                    textMsg.setText("哈哈" + i);
                    producer.send(textMsg);
                }
                long e = System.currentTimeMillis();
                System.out.println("发送消息成功");
                System.out.println(e - s);
                //即便生产者的对象关闭了，程序还在运行哦
                producer.close();
                
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
}
```

#### 二、ActiveMQ的应用

### 1、保证消息的成功处理

消息发送成功后，接收端接收到了消息。然后进行处理，但是可能由于某种原因，高并发也好，IO阻塞也好，反正这条消息在接收端处理失败了。而点对点的特性是一条消息，只会被一个接收端给接收，只要接收端A接收成功了，接收端B，就不可能接收到这条消息，如果是一些普通的消息还好，但是如果是一些很重要的消息，比如说用户的支付订单，用户的退款，这些与金钱相关的，是必须保证成功的，那么这个时候要怎么处理呢？

**我们可以使用  CLIENT_ACKNOWLEDGE  模式**

之前其实就有提到当创建一个session的时候，需要指定其事务，及消息的处理模式，当时使用的是 

```
session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
```

```
AUTO_ACKNOWLEDGE 
```

这一个代码的是，当消息发送给接收端之后，就自动确认成功了，而不管接收端有没有处理成功，而一旦确认成功后，就会把队列里面的消息给清除掉，避免下一个接收端接收到同样的消息。

那么，它还有另外一个模式，那就是 CLIENT_ACKNOWLEDGE

这行要写在接收端里面，不是写在发送端的

```
session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE);
```

 这行代码以后，如果接收端不确认消息，那么activemq将会把这条消息一直保留，直到有一个接收端确定了消息。

那么要怎么确认消息呢？

在接收端接收到消息的时候，调用javax.jms.Message的acknowledge方法

```
@Override
                public void onMessage(Message message) {
                    try {
                        //获取到接收的数据
                        String text = ((TextMessage)message).getText();
                        System.out.println(text);
                        //确认接收，并成功处理了消息
                        message.acknowledge();
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
```

这样，当消息处理成功之后，确认消息，如果不确定，activemq将会发给下一个接收端处理

 **注意：只在点对点中有效，订阅模式，即使不确认，也不会保存消息**

### 2、避免消息队列的并发

JMQ设计出来的原因，就是用来避免并发的，和沟通两个系统之间的交互。

### 主动接收队列消息

先看一下之前的代码:

```
            //实现一个消息的监听器
            //实现这个监听器后，以后只要有消息，就会通过这个监听器接收到
            consumer.setMessageListener(new MessageListener() {
                @Override
                public void onMessage(Message message) {
                    try {
                        //获取到接收的数据
                        String text = ((TextMessage)message).getText();
                        System.out.println(text);
                        //确认接收，并成功处理了消息
                        message.acknowledge();
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            });
```

之前的代码里面，实现了一个监听器，监听消息的传递，这样只要每有一个消息，都会即时的传递到程序中。

但是，这样的处理，在高并发的时候，因为它是被动接收，并没有考虑到程序的处理能力，可能会压跨系统，那要怎么办呢?

**答案就是把被动变为主动，当程序有着处理消息的能力时，主动去接收一条消息进行处理**

实现的代码如下:

```
　　　　　　if(当程序有能力处理){//当程序有能力处理时接收
                    Message receive = consumer.receive();　　　　　　　　　　　//这个可以设置超时时间，超过则不等待消息　　　　　　　　　　　　 recieve.receive(10000);
                    //其实receive是一个阻塞式方法，一定会拿到值的
                    if(null != receive){
                        String text = ((TextMessage)receive).getText();
                        receive.acknowledge();
                        System.out.println(text);
                    }else{
                        //没有值嘛
                        //
                    }
                }
             
```

**通过上面的代码，就可以让程序自已判断，自己是否有能力接收这条消息，如果不能接收，那就给别的接收端接收，或者等自己有能力处理的时候接收**

### 使用多个接收端

ActiveMQ是支持多个接收端的，如果当程序无法处理这么多数据的时候，可以考虑多个线程，或者增加服务器来处理。

### 3、消息有效期的管理

这样的场景也是有的，一条消息的有效时间，当发送一条消息的时候，可能希望这条消息在指定的时间被处理，如果超过了指定的时间，那么这条消息就失效了，就不需要进行处理了，那么我们可以使用ActiveMQ的设置有效期来实现

 代码如下:

```
            TextMessage msg = session.createTextMessage("哈哈");
            for(int i = 0 ; i < 100 ; i ++){
                //设置该消息的超时时间
                producer.setTimeToLive(i * 1000);
                producer.send(msg);
            }
```

 这里每一条消息的有效期都是不同的，打开ip:8161/admin/就可以查看到，里面的消息越来越少了。

 过期的消息是不会被接收到的。

 过期的消息会从队列中清除，并存储到ActiveMQ.DLQ这个队列里面，这个稍后会解释。 

#### 4、过期消息，处理失败的消息如何处理

过期的、处理失败的消息，将会被ActiveMQ置入“ActiveMQ.DLQ”这个队列中。

这个队列是ActiveMQ自动创建的。

如果需要查看这些未被处理的消息，可以进入这个队列中查看

```
//指定一个目的地，也就是一个队列的位置
destination = session.createQueue("ActiveMQ.DLQ");
```

这样就可以进入队列中，然后实现接口，或者通过receive()方法，就可以拿到未被处理的消息，从而保证正确的处理

#### 三、ActiveMQ的安全配置

###### 1、管理后台的密码设置

我们都知道，打开ip:8161/admin/ 就是activemq的管理控制台，它的默认账号和密码都是admin,在生产环境肯定需要更改密码的，这要怎么做呢？

在activemq/conf/jetty.xml中找到

```
   <pre name="code" class="html"> <bean id="securityConstraint" class="org.eclipse.jetty.util.security.Constraint">
        <property name="name" value="BASIC" />
        <property name="roles" value="admin" />
         <!-- 把这个改为true,当然，高版本的已经改为了true -->
        <property name="authenticate" value="true" />
  </bean>
```

高版本的已经默认成为了true。所以我们直接进行下一步即可

在activemq/conf/jetty-realm.properties文件中配置，打开如下

```
## ---------------------------------------------------------------------------
## Licensed to the Apache Software Foundation (ASF) under one or more
## contributor license agreements.  See the NOTICE file distributed with
## this work for additional information regarding copyright ownership.
## The ASF licenses this file to You under the Apache License, Version 2.0
## (the "License"); you may not use this file except in compliance with
## the License.  You may obtain a copy of the License at
##
## http://www.apache.org/licenses/LICENSE-2.0
##
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions and
## limitations under the License.
## ---------------------------------------------------------------------------

# Defines users that can access the web (console, demo, etc.)
# username: password [,rolename ...]
#用户名，密码，角色
admin: admin, admin
user: user, user
```

**注意：大家重点看倒数第二行，那里三个分别是用户名，密码，角色，其中admin角色是固定的**

###### 2、生产消费者的连接密码

**注意:activemq默认是不需要密码，生产消费者就可以连接的**

我们需要经过配置，才能设置密码，这一步在生产环境中一定要配置

找到activemq/conf/activemq.xml,并打开
在<broker>节点中，在<systemUsage>节点上面，增加如下的一个插件

```
<plugins>
             <simpleAuthenticationPlugin>
                 <users>
                     <authenticationUser username="${activemq.username}" password="${activemq.password}" groups="users,admins"/>
                 </users>
             </simpleAuthenticationPlugin>
</plugins>
```

这样就开启了密码认证
然后账号密码的配置在activemq/conf/credentials.properties文件中

打开这个文件如下

```
## ---------------------------------------------------------------------------
## Licensed to the Apache Software Foundation (ASF) under one or more
## contributor license agreements.  See the NOTICE file distributed with
## this work for additional information regarding copyright ownership.
## The ASF licenses this file to You under the Apache License, Version 2.0
## (the "License"); you may not use this file except in compliance with
## the License.  You may obtain a copy of the License at
##
## http://www.apache.org/licenses/LICENSE-2.0
##
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions and
## limitations under the License.
## ---------------------------------------------------------------------------

# Defines credentials that will be used by components (like web console) to access the broker

#账号
activemq.username=admin
#密码
activemq.password=123456
guest.password=password
```

## 三大消息中间件

### ActiveMQ

ActiveMQ是Apache出品，最流行的，能力强劲的开源消息总线。ActiveMQ是一个完全支持JMS1.1和J2EE1.4规范的JMS Provider实现，尽管JMS规范出台是很久的事情了，但JMS在当今的J2EE应用中仍然扮演着特殊的地位。

### RabbitMQ

RabbitMQ是一个开源的AMQP实现，服务端用Erlang语言编写，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

支持更多语言，基于AMQP规范

AMQP(advanced message queuing protocol)是一个提供统一消息服务的应用层标准协议，基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同开发语言等条件的限制 

### Kafka

Kafka是一种高吞吐量的分布式发布订阅消息系统，是一个分布式的、分区的、可靠的分布式日志存储服务，它通过一种独一无二的设计提供了一个消息系统的功能。

Kafka提供了类JMS的特性，但在设计实现上并不遵循JMS规范，Kafka对消息保存时根据Topic进行归类，发送消息者称为Producer,消息接受者称为Consumer,此外kafka集群有多个kafka实例组成，每个实例(server)称为broker。同时无论是kafka集群，还是producer和consumer都依赖于zookeeper集群保存一些meta信息，来保证系统可用性。

![这里写图片描述](https://img-blog.csdn.net/20171027100748177?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMzEyMzYzNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)