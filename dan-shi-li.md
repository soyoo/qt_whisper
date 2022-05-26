# qt application

#### Qt 之运行一个实例进程 <a href="#qt-zhi-yun-hang-yi-ge-shi-li-jin-cheng" id="qt-zhi-yun-hang-yi-ge-shi-li-jin-cheng"></a>

from: [http://www.aichengxu.com/dev/2553631.htm](http://www.aichengxu.com/dev/2553631.htm)

#### 简述 <a href="#jian-shu" id="jian-shu"></a>

发布程序的时候，我们往往会遇到这种情况：

1. 只需要用户运行一个实例进程
2. 用户可以同时运行多个实例进程

一个实例进程的软件有很多，例如：360、酷狗…

多个实例进程的软件也很多，例如：Visual Studio、Qt Ctretor、QQ…

下面我们来介绍下如何实现一个实例进程。

#### QSharedMemory <a href="#qsharedmemory" id="qsharedmemory"></a>

使用共享内存来实现，key值唯一，一般可以用组织名+应用名来确定。

首先，创建一个共享内存区，当第二个进程启动时，判断内存区数据是否建立，如果有，可以激活已打开的窗体，也可以退出。

当程序crash的时候，不能及时清除共享区数据，导致程序以后不能正常启动。

```cpp
int main(int argc, char **argv)
{
    QApplication app(argc, argv);

    QCoreApplication::setOrganizationName("Company");
    QCoreApplication::setApplicationName("AppName");
    QString strKey = QCoreApplication::organizationName() + QCoreApplication::applicationName();

    QSharedMemory sharedMemory(strKey);

    if (!sharedMemory.create(512, QSharedMemory::ReadWrite))
    {
        QMessageBox::information(NULL, QStringLiteral("提示"), QStringLiteral("程序已运行！"));
        exit(0);
    }

    MainWindow window;
    window.show();

    return app.exec();
}
```

#### QLocalServer <a href="#qlocalserver" id="qlocalserver"></a>

QSingleApplication.h

```cpp
#ifndef SINGLE_APPLICATION_H
#define SINGLE_APPLICATION_H

#include <QApplication>

class QLocalServer;

class QSingleApplication : public QApplication
{
    Q_OBJECT

public:
    explicit QSingleApplication(int argc, char **argv);
    // 判断进程是否存在
    bool isRunning();

private slots:
    void newLocalConnection();

private:
    QLocalServer *m_pServer;
    bool m_bRunning;
};

#endif // SINGLE_APPLICATION_H
```

QSingleApplication.cpp

```cpp
#include <QLocalSocket>
#include <QLocalServer>
#include <QFile>
#include <QTextStream>
#include "QSingleApplication.h"

QSingleApplication::QSingleApplication(int argc, char **argv)
    : QApplication(argc, argv),
      m_bRunning(false)
{
    QCoreApplication::setOrganizationName("Company");
    QCoreApplication::setApplicationName("AppName");
    QString strServerName = QCoreApplication::organizationName() + QCoreApplication::applicationName();

    QLocalSocket socket;
    socket.connectToServer(strServerName);

    if (socket.waitForConnected(500))
    {
        QTextStream stream(&socket);
        QStringList args = QCoreApplication::arguments();

        QString strArg = (args.count() > 1) ? args.last() : "";
        stream << strArg;
        stream.flush();
        qDebug() << "Have already connected to server.";

        socket.waitForBytesWritten();

        m_bRunning = true;
    }
    else
    {
        // 如果不能连接到服务器，则创建一个
        m_pServer = new QLocalServer(this);
        connect(m_pServer, SIGNAL(newConnection()), this, SLOT(newLocalConnection()));

        if (m_pServer->listen(strServerName))
        {
            // 防止程序崩溃，残留进程服务，直接移除
            if ((m_pServer->serverError() == QAbstractSocket::AddressInUseError) && QFile::exists(m_pServer->serverName()))
            {
                QFile::remove(m_pServer->serverName());
                m_pServer->listen(strServerName);
            }
        }
    }
}

void QSingleApplication::newLocalConnection()
{
    QLocalSocket *pSocket = m_pServer->nextPendingConnection();
    if (pSocket != NULL)
    {
        pSocket->waitForReadyRead(1000);

        QTextStream in(pSocket);
        QString strValue;
        in >> strValue;
        qDebug() << QString("The value is: %1").arg(strValue);

        delete pSocket;
        pSocket = NULL;
    }
}

bool QSingleApplication::isRunning()
{
    return m_bRunning;
}
```

使用方式

```cpp
int main(int argc, char **argv)
{
    QSingleApplication app(argc,argv);
    if (app.isRunning())
    {
        QMessageBox::information(NULL, QStringLiteral("提示"), QStringLiteral("程序已运行！"));
        exit(0);
    }

    MainWindow window;
    window.show();

    return app.exec();
}
```

#### QtSingleApplication <a href="#qtsingleapplication" id="qtsingleapplication"></a>

QSingleApplication位于qt-solution里面，并不包含在Qt库中，遵循 LGPL 协议。

文档、源码、示例见：[QtSingleApplication](https://github.com/qtproject/qt-solutions/tree/master/qtsingleapplication)

#### 任务列表 <a href="#ren-wu-lie-biao" id="ren-wu-lie-biao"></a>

运行程序时，遍历任务列表，查看是当前所有运行中的进程，如果当前进程位置在映射路径中可以找到，则说明程序已经运行，否则，未运行。

#### 更多参考 <a href="#geng-duo-can-kao" id="geng-duo-can-kao"></a>

[SingleApplication](https://github.com/itay-grudev/SingleApplication)

[Single App Instance](http://berenger.eu/blog/c-qt-singleapplication-single-app-instance/)
