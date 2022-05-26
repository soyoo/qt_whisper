# qt network reconnect

qt 实现 socket 断线重连机制

from: [http://blog.csdn.net/qq\_35488967/article/details/69388213](http://blog.csdn.net/qq\_35488967/article/details/69388213)

#### 简述 <a href="#jian-shu" id="jian-shu"></a>

* 创建 Thread 类 继承 QThread，实现用单独的线程接收 socket 数据。
* 当 socket 与主机断开时，自动触发 OnDisConnect() 函数，从而在 run() 中执行自动重连代码段。
* 想主动断开 socket 连接时，把 m\_isThreaStopped 设置为 true 即可。

#### 类的源码 <a href="#lei-de-yuan-ma" id="lei-de-yuan-ma"></a>

Thread.h

```cpp
#ifndef THREAD_H
#define THREAD_H

#include <QThread>

class QTcpSocket;
class QTextCodec;

class Thread : public QThread
{
    Q_OBJECT

public:
    Thread(QObject *parent);
    ~Thread();

    void startThread(const QString& ip, int port);
    void stopThreaad();

protected:
    virtual void run();

signals:
    void sendMSg(QString msg);

protected slots:
    void onConnect();
    void onDisConnect();
    void onReadMsg();

private:
    QTcpSocket* m_TcpSocket;

    bool m_isThreaStopped;
    bool m_isOkConect;
    QString m_QStrSocketIp;
    int m_nSockPort;
    QByteArray m_datagram;
};

#endif // THREAD_H
```

// Thread.cpp

```cpp
#include "Thread.h"
#include <QTcpSocket>
#include <QTextCodec>
Thread::Thread(QObject *parent)
    : QThread(parent)
    , m_TcpSocket(NULL)
    , m_isThreaStopped(false)
    , m_isOkConect(false)
{
}

Thread::~Thread()
{
}

//线程的run函数
void Thread::run()
{
    while (!m_isThreaStopped)
    {
        //检测客户端 socket指针是否为空
        if (!m_TcpSocket)
        {
            m_TcpSocket = new QTcpSocket(this);
            connect(m_TcpSocket, SIGNAL(readyRead()), this, SLOT(onReadMsg()));
            connect(m_TcpSocket, SIGNAL(connected()), this, SLOT(onConnect()));
            connect(m_TcpSocket, SIGNAL(disconnected()), this, SLOT(onDisConnect()));
        }
        if (!m_isOkConect)
        {
            m_TcpSocket->connectToHost(m_QStrSocketIp, m_nSockPort);
            //等待连接。。。延时三秒，三秒内连不上返回false
            m_isOkConect = m_TcpSocket->waitForConnected(3000);
        }
        if (!m_isOkConect)
        {
            continue;
        }
        m_TcpSocket->waitForReadyRead(5000);
    }
}

void Thread::onDisConnect()
{
    //socket一旦断开则自动进入这个槽函数
    //通过把 m_isOkConect 设为false，在socket线程的run函数中将会重新连接主机
    m_isOkConect = false;
}

void Thread::startThread(const QString& ip, int port)
{
    m_QStrSocketIp = ip;
    m_nSockPort = port;
    m_isThreaStopped = false;
    start();
}

void Thread::stopThreaad()
{
    m_isThreaStopped = true;
}

void Thread::onConnect()
{
    //已连接
}

void Thread::onReadMsg()
{
    while (m_TcpSocket->bytesAvailable() > 0)
    {
        m_datagram.clear();
        m_datagram.resize(m_TcpSocket->bytesAvailable());
        m_TcpSocket->read(m_datagram.data(), m_datagram.size());
        QString string = QString::fromLocal8Bit(m_datagram);
        emit sendMSg(string);
    }
}
```

#### 类的使用 <a href="#lei-de-shi-yong" id="lei-de-shi-yong"></a>

main.cpp

```cpp
#include "TestFun.h"
#include <QtWidgets/QApplication>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    TestFun w;
    w.show();
    return a.exec();
}
```

主界面类，线程接受到的数据在这个主界面上显示出来。

testFun.h

```cpp
#ifndef TESTFUN_H
#define TESTFUN_H

#include <QtWidgets/QMainWindow>
#include "ui_TestFun.h"

class Thread;

class TestFun : public QMainWindow
{
    Q_OBJECT

public:
    TestFun(QWidget *parent = Q_NULLPTR);

private slots:
    void on_btnConnect_clicked();
    void dataReaceave(QString msg);

private:
    Ui::TestFunClass ui;
    Thread* socketThread;
};

#endif //TESTFUN_H
```

testFun.cpp

```cpp
#include "TestFun.h"
#include <QTime>
#include "Thread.h"

TestFun::TestFun(QWidget *parent)
    : QMainWindow(parent)
    , socketThread(NULL)
{
    ui.setupUi(this);

    // 一定要先 new 再 connnect，否则无法接收到数据。
    socketThread = new Thread(this);
    connect(socketThread, SIGNAL(sendMSg(QString)), this, SLOT(dataReaceave(QString)), Qt::QueuedConnection);
    on_btnConnect_clicked();
}

// 按钮槽函数，按钮是用 .ui 文件写的，我就不上传了。
void TestFun::on_btnConnect_clicked()
{
    socketThread->startThread("127.0.0.1", 8089);
}

void TestFun::dataReaceave(QString msg)
{
    // 主线程在界面上显示数据
    ui.textEdit->setText(msg);
}
```

testFun.ui

![这里写图片描述](http://img.blog.csdn.net/20170827232009182?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzU0ODg5Njc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 效果图 <a href="#xiao-guo-tu" id="xiao-guo-tu"></a>

![这里写图片描述](http://img.blog.csdn.net/20170827233458970?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzU0ODg5Njc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到当我们点服务器的“断开”按钮后，虽然在一瞬间断开了，但是客户端马上自动重连，恢复了连接。
