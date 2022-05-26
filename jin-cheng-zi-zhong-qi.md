# How to make an Application restartable

from: [http://wiki.qt.io/How\_to\_make\_an\_Application\_restartable](http://wiki.qt.io/How\_to\_make\_an\_Application\_restartable)

If you are in the need to make an application restartable depending on the user interaction you have to follow these steps:

**Create an exit code that represents your reboot/restart event**

It is a good idea to define this code as a static variable in your main window:

```cpp
static int const EXIT_CODE_REBOOT;
```

and initialize it with a value:

```cpp
static int const MainWindow::EXIT_CODE_REBOOT = -123456789; 
```

**Define a Slot in your Application**

Next define a slot that will exit the application using the reboot code:

```cpp
void MainWindow::slotReboot()
{
  qDebug() << "Performing application reboot..." << endl;
  qApp->exit(MainWindow::EXIT_CODE_REBOOT);
}
```

**Create a QAction to handle the Reboot**

Create an action that will consume the above slot in order to exit with the reboot code. Something like the following will works:

```cpp
actionReboot = new QAction(this);
actionReboot->setText(tr("Restart"));
actionReboot->setStatusTip(tr("Restarts the application"));
connect(actionReboot, SIGNAL(triggered()), this, SLOT(slotReboot()));
```

**Modify the Application Cycle**

The last step is to modify the application main function to handle the new cycle that will allow rebooting:

```cpp
int main(int argc, char **argv)
{
  int currentExitCode = 0;

  do
  {
    QApplication a(argc, argv);
    MainWindow w;
    w.show();
    currentExitCode = a.exec();
  } while (currentExitCode == MainWindow::Exit_CODE_REBOOT);

  return currentExitCode;
}
```

**How to trigger actionReboot's triggered() signal by a QPushButton**

Just connect QPushButton's clicked() signal with actionReboot's triggered() signal:

```cpp
connect(ui->pushButton, SIGNAL(clicked()), actionReboot, SIGNAL(triggered()));
```
