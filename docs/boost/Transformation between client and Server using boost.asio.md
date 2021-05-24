# 0 序言
boost.asio提供了同步、异步读写数据，以及网络通信的工具。从应用层抽象了网络结构，方便程序员快速实现网络程序。

本篇将介绍如何使用boost.asio提供的接口，来实现一个server与client之间通信的例子，构建一个聊天室。其实现了以下通信的基本功能：
1. server监听ip:port。
2. client向server发送信息。
3. server接收信息，并向所有的client广播信息。

# 1 存储数据的类
通常我们采用如下两种方式，给数据设定边界，本篇文章我们采用第一种方式。
1. 在数据传输开始，给出后面将要传输数据的长度。
2. 通过特殊字符，设定传输数据的边界。

下面代码展示了传输数据的定义。

```
// chatMessage.h
#include <cstdio>
#include <cstdlib>
#include <cstring>

class ChatMessage {

public:
    enum {
        headerLen = 4,
        maxBodyLen = 512,
    };

public:
    ChatMessage() : m_bodyLen(0) {}

    const char* data() const { return m_data; }
    char* data() { return m_data; }

    const char* body() const { return m_data + headerLen; }
    char* body() { return m_data + headerLen; }

    size_t wholeLen() const { return headerLen + m_bodyLen; }
    size_t bodyLen() const { return m_bodyLen; }
    void setBodyLen(size_t newLen) { m_bodyLen = newLen > maxBodyLen ? maxBodyLen : newLen; }

    void encodeHeader()
    {
        char hd[headerLen+1] = "";
        std::sprintf(hd, "%4d", static_cast<int>(m_bodyLen));
        memcpy(m_data, hd, headerLen);
    }
    bool decodeHeader()
    {
        char hd[headerLen+1]="";
        strncpy(hd, m_data, headerLen);
        m_bodyLen = atoi(hd);
        if (m_bodyLen > maxBodyLen) {
            m_bodyLen = 0;
            return false;
        }
        return true;
    }
private:
    char m_data[headerLen + maxBodyLen];
    size_t m_bodyLen;
};
```

上面定义了类ChatMessage，其以数组形式，最多可存储1028+4个byte的数据。其中：
1. m_data为数据的缓冲区，用于存放写入网络和从网络读取的数据。
1. m_data的前4个字节存储数据的长度，通过decodeHeader()和encodeHeader()方法来将char[0:3]和int进行转换。
2. m_data从第5个字节开始存储实际数据，并使用m_bodyLen来存储数据主体的长度。

# 2 Server端
server端涉及到3个类型：接收message的server程序，讨论者在server端的代表，和管理讨论者的容器。

### 1 讨论者在server端的代表
如下为定义的讨论者。因为不同平台下，数据传输方式不同，因此，我们首先定义了传输数据的deliver()接口，抽象数据的具体传输方式。

```
// chatServer.h
class ChatParticipant {
  public:
    virtual ~ChatParticipant() {}
    virtual void deliver(const ChatMessage& msg) = 0;
};
typedef boost::shared_ptr<ChatParticipant> ChatParticipantPtr;
```

下面是ChatParticipant的具体实现类，其为讨论者在聊天室中的代理人。通过该类，server接收到client发送的信息，并将获得的消息，通过各个讨论者的代理人，转发给所有的client。

```
// chatServer.h
class ChatRoom;
class ChatSession : public ChatParticipant, public boost::enable_shared_from_this<ChatSession> {

  public:
    ChatSession(boost::asio::io_context& ioContext, ChatRoom& room);

    void start();
    virtual void deliver(const ChatMessage& msg);

    void handleRdHeader(const boost::system::error_code& error);
    void handleRdBody(const boost::system::error_code& error);
    void handleWt(const boost::system::error_code& error);

    tcp::socket& socket();

  private:
    tcp::socket m_socket;
    ChatRoom& m_chatRoom; // 对应的聊天室
    ChatMessage m_rdMsg;  // 从client接收到的数据
    ChatMessageQueue m_wtMsgQueue; // 等待发送给client的数据
};
typedef boost::shared_ptr<ChatSession> ChatSessionPtr;
```

下面为ChatSession的具体实现。

```
ChatSession::ChatSession(boost::asio::io_context &ioContext, ChatRoom &room)
    : m_socket(ioContext), m_chatRoom(room) {}

tcp::socket& socket() { return m_socket; }
```

start()将自己传给ChatRoom，表示愿意主动接收聊天室的信息，并向聊天室发送消息。boost::asio::async_read()分别传入三个参数：抽象物理连接的socket，接收从socket读取数据的缓存，完成数据接收后需要做善后处理的回调函数。所以，异步流程就是：
1. 程序等待socket传入数据进来。——不知道什么时候才能到来，所以不会阻塞
2. 数据到达，开始从socket读取数据到缓存。
3. 数据读取结束，调用善后回调函数，做进一步处理。

```
void ChatSession::start()
{
    m_chatRoom.join(shared_from_this());
    boost::asio::async_read(m_socket,
            boost::asio::buffer(m_rdMsg.data(), ChatMessage::headerLen),
            boost::bind(&ChatSession::handleRdHeader, shared_from_this(), boost::asio::placeholders::error));
}
```

deliver()方法首先将需要传给client的数据放到队列里面。然后调用boost::asio::async_write()方法将数据写入到socket中。同样涉及三个参数：抽象物理连接的socket, 存储待发送数据的缓存空间，完成数据发送后需要做善后处理的回调函数。所以，异步流程就是：
1. 程序等待socket是否已经准备好开始传输数据。——不知道什么时候端口才能空闲，所以不会阻塞
2. socket准备就绪，开始从缓存提取数据并发送到物理网络。
3. 发送完成后，调用善后处理函数，做进一步处理。

```
void ChatSession::deliver(const ChatMessage &msg)
{
    m_wtMsgQueue.push_back(msg);
    boost::asio::async_write(m_socket,
                             boost::asio::buffer(m_wtMsgQueue.front().data(), m_wtMsgQueue.front().data().wholeLen()),
                             boost::bind(&ChatSession::handleWt, shared_from_this(), boost::asio::placeholders::error)
                             );
}
```

下面是三个善后处理函数。需要注意：
1. 在读取过程start()中，我们只是读取了到来数据的头部，即到来数据的长度信息。
2. 得到数据的长度信息后，交由handlerRdHeader()对头部信息进行处理。
3. 已知了数据主体的长度，我们才开始真正读取数据主体，这个工作交由handleRdBody()来处理。
如果产生错误，讨论者将被自动请出聊天室，因为数据传输不畅。

```
void ChatSession::handleRdHeader(const boost::system::error_code &error)
{
    if (!error && m_rdMsg.decodeHeader()) {
        boost::asio::async_read(m_socket,
                                boost::asio::buffer(m_rdMsg.body(), m_rdMsg.bodyLen()),
                                boost::bind(&ChatSession::handleRdBody, shared_from_this(), boost::asio::placeholders::error)
                                );
    } else {
        m_chatRoom.leave(shared_from_this());
    }
}
```

当我们完成了当前到来数据的读取工作时，后面可能还会由数据到来，所以，继续调用异步方法boost::asio::async_read()来监听socket，如果有数据，继续读取。

```
void ChatSession::handleRdBody(const boost::system::error_code &error)
{
    if (error) {
        m_chatRoom.leave(shared_from_this());
        return;
    }
    m_chatRoom.deliver(m_rdMsg);
    boost::asio::async_read(m_socket,
                            boost::asio::buffer(m_rdMsg.data(), ChatMessage::headerLen),
                            boost::bind(&ChatSession::handleRdHeader, shared_from_this(),
                                        boost::asio::placeholders::error)
                            );
}
```

当我们将队头数据写到socket后，队列中可能仍然有需要写入socket的数据，因此，依然调用boost::asio::async_write，来等待socket是否准备好，然后继续向网络里写入数据。

```
void ChatSession::handleWt(const boost::system::error_code &error)
{
    if (error) {
        m_chatRoom.leave(shared_from_this());
        return;
    }
    m_wtMsgQueue.pop_front();
    if (!m_wtMsgQueue.empty()) {
        boost::asio::async_write(m_socket,
                                 boost::asio::buffer(m_wtMsgQueue.front().data(), m_wtMsgQueue.front().wholeLen()),
                                 boost::bind(&ChatSession::handleWt, shared_from_this(), boost::asio::placeholders::error)
                                 );
    }
}
```

### 2 聊天室
聊天室实际上是一个容器，其盛装所有的“讨论者代表”。讨论者进入聊天室的唯一意义，在于其是否可以向聊天室其他人发送信息，以及接收聊天室其他人的信息。所以，下面方法主要用于控制聊天室的讨论者。

join()和leave()方法由“讨论者代理人”自行调用，表示加入和退出聊天室。deliver()为广播信息，当某个client发送信息后，其信息将会在整个聊天室进行广播。

由于我们采用异步方式发送信息，无法保证所有信息能立即发送给所有client，所以ChatMessageQueue用于缓存没有广播出去的ChatMessage。

```
class ChatRoom {
  public:
    void join(ChatParticipantPtr participant);
    void leave(ChatParticipantPtr participant);
    void deliver(const ChatMessage msg);

  private:
    enum { maxMsgNum = 100, };

  private:
    std::set<ChatParticipantPtr> m_participateSet;
    ChatMessageQueue m_msgQueue;
};
```

下面函数添加了一个“讨论者代理人”，并将所有还存在队列中的消息，发送给刚刚加入的代理人。

```
void ChatRoom::join(ChatParticipantPtr participant)
{
    m_participateSet.insert(participant);
    std::for_each(m_msgQueue.begin(), m_msgQueue.end(),
        boost::bind(&ChatParticipant::deliver, participant, _1);
}
```

deliver()将需要发送的消息广播给所有的的“讨论者代理人”。

```
void ChatRoom::leave(ChatParticipantPtr participant) { m_participateSet.erase(participant); }

void ChatRoom::deliver(const ChatMessage msg)
{
    m_msgQueue.push_back(msg);
    while (m_msgQueue.size() > maxMsgNum) {
        m_msgQueue.pop_front();
    }

    std::for_each(m_participateSet.begin(), m_participateSet.end(),
                  boost::bind(&ChatParticipant::deliver, _1, boost::ref(msg)));
}
```

### 3 server程序
最后，就是server程序。其监听指定接口，并接受client接入chatRoom的请求。类定义如下.
1. boost::asio::io_context表示系统平台提供的环境接口，所有关于io的操作，最终都由该类对象来执行。
2. boost::asio::ip::tcp::endpoint代表一个端口，有ip:port来唯一确定和描述。
3. boost::asio::ip::tcp::acceptor表示监听对象，其监听指定endpoint。

```
class ChatServer {
  public:
    ChatServer(boost::asio::io_context& ioContext, const tcp::endpoint& endpoint);
    void startAccept();
    void handleAccept(ChatSessionPtr session, const boost::system::error_code& error);

  private:
    boost::asio::io_context& m_ioContext;
    tcp::acceptor m_acceptor;
    ChatRoom m_chatRoom;
};

typedef boost::shared_ptr<ChatServer> ChatServerPtr;
typedef boost::list<ChatServerPtr> ChatServerList;
```

m_acceptor为异步接受每次请求。当某次接入请求到达时，新加入的session将会被加入到chatroom中。需要注意：
1. server只负责将“讨论者代理人”接入到聊天室中，而真正保持让所有“代理人”进行通信的工作，由聊天室来承担。
2. server在接受请求连接后，boost::asio::ip::tcp::acceptor::async_accept将会调用善后处理函数handleAccept()，该函数会启动session的监听动作，保证session开始读取数据，并继续等待下一个client的接入请求。

```
ChatServer::ChatServer(boost::asio::io_context &ioContext, const tcp::endpoint &endpoint)
    : m_ioContext(ioContext), m_acceptor(ioContext, endpoint) {  startAccept(); }

void ChatServer::startAccept()
{
    ChatSessionPtr newSession(new ChatSession(m_ioContext, m_chatRoom));
    m_acceptor.async_accept(newSession->socket(),
                            boost::bind(&ChatServer::handleAccept, this,
                            newSession, boost::asio::placeholders::error)
                            );
}

void ChatServer::handleAccept(ChatSessionPtr session, const boost::system::error_code &error)
{
    if (!error) {
        session->start();
    }
    startAccept();
}
```

### 4 启动服务
最后，我们将在main()中启动上面定义的服务器。

```
int main(int ac, char** av)
{
    try {

        if (ac < 2) {
            std::cerr << "Usage : server <port> \n";
            return 1;
        }

        boost::asio::io_context ioContext;
        ChatServerList serverList; // 允许多个端口进行监听，即创建多个聊天室，每个聊天室对应一个端口
        for (int index = 1; index < ac; ++index) {
            tcp::endpoint endpoint(tcp::v4(), atoi(av[index]));
            ChatServerPtr server(new ChatServer(ioContext, endpoint));
            serverList.push_back(server);
        }
        ioContext.run(); // 启动平台的接口服务

    } catch (std::exception& e) {
        std::cerr << "Exception : " << e.what() << std::endl;
    }
    return 0;
}
```

# 3 client端
client端需要做如下三件事来保证在网络上顺畅传递信息；
1. 连接到服务器。
2. 从server读取数据。
(1) 读取message头部
(2) 读取信息主体
3. 向server写入数据

下面为client的定义。在执行上面三个过程中，我们需要注意，由于受到各种因素的影响，真正与服务器构建连接、读取数据、写入数据的时间点都是很难预测的。所以，异步通信保证了只有某个事件发生时，上面的三个操作才会真正发生。即：
1. handleconn()只有与server建立连接，才会执行。
2. handleRdHeader()只有读取了数据头，才会执行。
3. handleRdBody()只有读取了数据主体，才会执行。
4. handleWt()只有将数据写入网络，才会执行。

```
typedef std::deque<ChatMessage> ChatMessageQueue;

class ChatClient {

public:
    ChatClient(boost::asio::io_context& ioContext, const boost::asio::ip::tcp::resolver::result_type& endPoint);
    void write(const ChatMessage& msg);
    void close();

private:
    void handleConn(const boost::system::error_code& error);

    void handleRdHeader(const boost::system::error_code& error);
    void handleRdBody(const boost::system::error_code& error);

    void doWrite(ChatMessage& msg);
    void handleWt(const boost::system::error_code& error);

    void doClose();

private:
    boost::asio::io_context& m_ioContext;
    boost::asio::ip::tcp::socket m_socket;
    ChatMessage m_rdMsg;
    ChatMessageQueue m_wtMsg;
};
```

 boost::asio::async_connect为异步连接，保证只有在client与server连接成功后，才能开始调用handlConn()方法。

```
ChatClient::ChatClient(boost::asio::io_context &ioContext,
            const boost::asio::ip::tcp::resolver::result_type &endPoint)
    : m_ioContext(ioContext), m_socket(ioContext)
{
    boost::asio::async_connect(m_socket, endPoint,
            boost::bind(&ChatClient::handleConn, this, boost::asio::placeholders::error));
}
```

boost::asio::post请求系统执行指定的函数。此处执行doWrite()方法。

```
void ChatClient::write(const ChatMessage &msg)
{
    boost::asio::post(m_ioContext, boost::bind(&ChatClient::doWrite, this, msg));
}

void ChatClient::close() // 关闭连接
{
    boost::asio::post(m_ioContext, boost::bind(&ChatClient::doClose, this));
}
```

当client与server连接成功后，其开始异步等待监听接口的数据。调用boost::asio::async_read()，并将handleRdHeader和handleRdBody作为善后处理函数，分别读取数据的长度和主体。

当处理完当前的数据后，client调用boost::asio::async_read()继续等待到来的数据。

```
void ChatClient::handleConn(const boost::system::error_code &error)
{
    if (error) return;

    boost::asio::async_read(m_socket,
                boost::asio::buffer(m_rdMsg.data(), ChatMessage::headerLen),
                boost::bind(&ChatClient::handleRdHeader, this, boost::asio::placeholders::error)
                );
}


void ChatClient::handleRdHeader(const boost::system::error_code &error)
{
    if (!error && m_rdMsg.decodeHeader()) {
        boost::asio::async_read(m_socket,
                boost::asio::buffer(m_rdMsg.data(), m_rdMsg.bodyLen()),
                boost::bind(&ChatClient::handleRdBody, this, boost::asio::placeholders::error)
                                );
    } else {
        doClose();
    }
}

void ChatClient::handleRdBody(const boost::system::error_code &error)
{
    if (error) {
        doClose();
        return;
    }
    std::cout.write(m_rdMsg.body(), m_rdMsg.bodyLen());
    std::cout << std::endl;

    boost::asio::async_read(m_socket,
                boost::asio::buffer(m_rdMsg.data(), ChatMessage::headerLen),
                boost::bind(&ChatClient::handleRdHeader, this, boost::asio::placeholders::error)
                );
}
```

当client需要向网络中写数据时，首先将数据放到缓存队列中，并使用函数boost::asio::async_write()异步等待数据写入网络。数据传输成功后，调用回调函数handleWt()进行删除处理。需要注意，只有在缓冲队列中还有数据未被发送时，才继续调用boost::asio::async_write()。

```
void ChatClient::doWrite(ChatMessage &msg)
{
    m_wtMsg.push_back(msg);

    boost::asio::async_write(m_socket,
                boost::asio::buffer(m_wtMsg.front().data(), m_wtMsg.front().wholeLen()),
                boost::bind(&ChatClient::handleWt, this, boost::asio::placeholder::error)
                             );
}

void ChatClient::handleWt(const boost::system::error_code &error)
{
    if (error) {
        doClose();
        return;
    }
    m_wtMsg.pop_front();
    if (!m_wtMsg.empty()) {
        boost::asio::async_write(m_socket,
                boost::asio::buffer(m_wtMsg.front().data(), m_wtMsg.front().wholeLen()),
                boost::bind(&ChatClient::handleWt, this, boost::asio::placeholders::error)
                                 );
    }
}


void ChatClient::doClose()
{
    m_socket.close();
}
```

最后，给出main()函数。
1. resolver用来解析ip:port,从而构建出server的端点endpoint，使client创建socket，连接server.
2. boost::thread创建新线程，来保证ChatClien对socket进行监听。而主线程供用户输入信息，发向服务器。
3. 当用户不再发送信息时，表示自动退出聊天室，连接被关闭，线程结束。

```
int main(int ac, char** av)
{
    try {

        if (ac != 3) {
            std::cerr << "Usage: client <ip> <port>" << std::endl;
            return 1;
        }

        boost::asio::io_context ioContext;

        tcp::resolver resolver(ioContext);
        tcp::resolver::result_type endpoint = resolver.resolve(av[1], av[2]);

        ChatClient client(ioContext, endpoint);

        boost::thread thread(boost::bind(&boost::asio::io_context::run, &ioContext));

        char line[ChatMessage::maxBodyLen + 1];
        while (std::cin.getline(line, ChatMessage::maxBodyLen + 1)) {
            ChatMessage msg;
            msg.setBodyLen(strlen(line));
            memcpy(msg.body(), line, msg.bodyLen());
            msg.encodeHeader();
            client.write(msg);
        }
        client.close();
        thread.join();

    } catch (std::exception& e) {
        std::cerr << "Exception" << e.what() << std::endl;
    }
    return 0;
}
```
