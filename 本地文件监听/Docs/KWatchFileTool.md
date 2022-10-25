# KWatchFileTool-文件监听工具

------

[TOC]



## 1.KWatchFileTool界面结构介绍

### 1.1界面的各功能介绍

![1](image\1.png)

### 1.2快捷键介绍

|          功能          |   按键   |
| :--------------------: | :------: |
|        开始监听        |   F1键   |
|          查询          |   F2键   |
|         最小化         |   F3键   |
|        退出程序        |  ESC键   |
|      保存当前输出      | Ctrl+O键 |
| 清空当前所有输出框输出 | Ctrl+S键 |



### 1.3KWatchFileTool项目主要分为六个部分

线程：ktask.h

选择框：kcanvaschoosebox.h

数据库：kdatabase.h

关键字查询：kinquire.h

输出框图：kshowdatawindow.h

菜单栏：kmenubar.h

### 1.4本项目的实现思路

![2](image\2.png)

### 1.5本项目的实现亮点

- 创建快捷键方便使用键盘辅助操作
- 数据库设置手动和自动提交两种模式更加人性化
- 消息提醒使用图标闪烁提示，不占用桌面使用范围

### 1.6本项目的难点

- 监听信息并且实时打印输出
- movetothread（）函数后，取消线程的循环阻塞
- 主线程和子线程的初始化界面分离
- 关键字查找和排序，处理数据库中数据的相同部分

## 2.各部分主要代码详细介绍

#### 2.1线程：ktask.h

将线程放入到KTask对象中去执行任务

```c++
//初始化线程
void KWatchFileTool::initThread()
{
    m_pTask = new KTask;//new 一个新KTask对象

    m_pKthread = new QThread;//new一个新子线程

    m_pTask->moveToThread(m_pKthread);//将线程放入到KTask中

    (void)connect(this, SIGNAL(taskOneStart()), m_pTask, SLOT(taskOne()));//接收开始信号，连接槽函数
    (void)connect(this, SIGNAL(deleteTask()), this, SLOT(setTrue()));//暂停
    (void)connect(this, SIGNAL(quitBlock()), this, SLOT(pauseSubThread()));//删除线程
    (void)connect(m_pTask, SIGNAL(deleteThread()), this, SLOT(deleteThreads()));//删除线程
    //打印监听文件信息
    (void)connect(m_pTask, SIGNAL(printf(QString)), m_pTextEdit, SLOT(printfAddfile(QString)));
    (void)connect(m_pTask, SIGNAL(printfDelete(QString)), m_pTextEdit, SLOT(printfDeletefile(QString)));
    (void)connect(m_pTask, SIGNAL(printfModi(QString)), m_pTextEdit, SLOT(printfModifile(QString)));
    (void)connect(m_pTask, SIGNAL(printfRename(QString)), m_pTextEdit, SLOT(printfRenamefile(QString)));

    //按钮组的状态变化选择
    (void)connect(m_pCanvasChooseBox, SIGNAL(event(int, bool)), m_pTask, SLOT(Add(int, bool)));
    (void)connect(m_pCanvasChooseBox, SIGNAL(event(int, bool)), m_pTask, SLOT(Delete(int, bool)));
    (void)connect(m_pCanvasChooseBox, SIGNAL(event(int, bool)), m_pTask, SLOT(Modi(int, bool)));
    (void)connect(m_pCanvasChooseBox, SIGNAL(event(int, bool)), m_pTask, SLOT(Rename(int, bool)));

    //是否监听子目录
    (void)connect(m_pCheckBox, SIGNAL(stateChanged(int)), m_pTask, SLOT(classBox()));

}
```



KTask中的执行任务函数

1.绑定文件句柄

```c++
	hDirectory = CreateFile(KGlobalData::getGlobalDataIntance()->getFilePath().toStdWString().c_str(), 
        				GENERIC_READ | GENERIC_WRITE | FILE_LIST_DIRECTORY,
        				FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,
        				NULL, 
        				OPEN_EXISTING, 
        				FILE_FLAG_BACKUP_SEMANTICS |FILE_FLAG_OVERLAPPED, 
        				NULL);//获取文件路径，绑定文件句柄

    if (hDirectory == INVALID_HANDLE_VALUE)
    {
        DWORD dwErr = GetLastError();//错误
        return;
    }
```

2.字符处理

```c++
void KTask::W2C(wchar_t* pwszSrc, int iSrcLen, char* pszDest, int iDestLen)
{
    RtlZeroMemory(pszDest, iDestLen);//初始化内存空间为全0

    // 宽字节字符串转多字节字符串
    WideCharToMultiByte(CP_ACP,
        0,
        pwszSrc,
        (iSrcLen / 2),
        pszDest,
        iDestLen,
        NULL,
        NULL);

}
```

3.阻塞监听文件变化

```c++
    do
    {
        RtlZeroMemory(pFileNotifyInfo, dwBufferSize);//初始化内存空间为全0

        // 设置监控目录，监控文件变化
        bRet = ReadDirectoryChangesW(hDirectory,
            pFileNotifyInfo,
            dwBufferSize,
            TRUE,
            FILE_NOTIFY_CHANGE_FILE_NAME |
            FILE_NOTIFY_CHANGE_ATTRIBUTES |
            FILE_NOTIFY_CHANGE_LAST_WRITE,
            &dwRet,
            NULL,
            NULL);

        if (FALSE == bRet)
        {
            break;//如果监听失败，退出程序
        }

        if (m_flag)
        {
            emit deleteThread();//发送删除线程信号
            break;//暂停，跳出循环
        }

        // 将宽字符转换成窄字符
        W2C((wchar_t*)(&pFileNotifyInfo->FileName), pFileNotifyInfo->FileNameLength, szTemp, MAX_PATH);

        //判断执行操作种类
        if (pFileNotifyInfo->Action == FILE_ACTION_ADDED && m_addkey)
        {
            // 新增文件
            m_FileName = QString::fromWCharArray(pFileNotifyInfo->FileName);
            emit printf(m_FileName);//发送监听到的文件名
        }

        if (pFileNotifyInfo->Action == FILE_ACTION_REMOVED && m_deletekey)
        {
            //删除文件
            m_FileName = QString::fromWCharArray(pFileNotifyInfo->FileName);
            emit printfDelete(m_FileName);//发送监听到的文件名
        }

        if (pFileNotifyInfo->Action == FILE_ACTION_MODIFIED && m_modikey)
        {
            //修改文件
            m_FileName = QString::fromWCharArray(pFileNotifyInfo->FileName);
            emit printfModi(m_FileName);//发送监听到的文件名
        }

        if (pFileNotifyInfo->Action == FILE_ACTION_RENAMED_OLD_NAME && m_renamekey)
        {
            //重命名文件
            m_FileName = QString::fromWCharArray(pFileNotifyInfo->FileName);
            emit printfRename(m_FileName);//发送监听到的文件名
        }

        KGlobalData::getGlobalDataIntance()->setFileName(m_FileName);//更新监听文件名

        //获取文件后缀
        QStringList str = KGlobalData::getGlobalDataIntance()->getFileName().split(".");//在.处断开文件名，获取后缀
        QString suffix = str.at(str.size() - 1);//获取文件名后缀
        KGlobalData::getGlobalDataIntance()->setSuffixName(suffix);//更新监听文件名后缀

        //存储文件后缀到全局
        QStringList tmp = KGlobalData::getGlobalDataIntance()->getSuffixList();//文件名后缀链表
        bool is = true;

        if (tmp.isEmpty())//如果为空
        {
            tmp.append(suffix);//直接加入链表
        }
        else//否则
        {
            for (int i = 0; i < tmp.size() - 1; ++i)
            {
                if (tmp.at(i) == suffix)//如果新的后缀和之前的一样
                    is = false;//设置flag为false
            }
            if (is)
                tmp.append(suffix);//如果不是false，加入后缀到文件后缀链表
        }
        KGlobalData::getGlobalDataIntance()->setSuffixList(tmp);//设置到全局变量中


    } while (bRet && !m_flag);//循环判断条件

    // 关闭句柄, 释放内存
    CloseHandle(hDirectory);
    delete[] pBuf;
    pBuf = NULL;
    return ;
```



#### 2.2选择框：kcanvaschoosebox.h

1.checkbox状态改变，发送信号到处理槽函数

```c++
//checkbox改变状态，发送给槽函数
void KCanvasChooseBox::changeStatus()
{
	(void)connect(m_pFileCreate, SIGNAL(stateChanged(int)), this, SLOT(classifyBox(int)));//文件增加按钮，状态改变
	(void)connect(m_pFileDelete, SIGNAL(stateChanged(int)), this, SLOT(classifyBox(int)));//文件删除按钮，状态改变
	(void)connect(m_pFileChange, SIGNAL(stateChanged(int)), this, SLOT(classifyBox(int)));//文件改变按钮，状态改变
	(void)connect(m_pFileRename, SIGNAL(stateChanged(int)), this, SLOT(classifyBox(int)));//文件重命名按钮，状态改变
	(void)connect(m_pFileRemove, SIGNAL(stateChanged(int)), this, SLOT(classifyBox(int)));//文件移动按钮，状态改变

}
```



2.信号处理槽函数

```c++
void KCanvasChooseBox::classifyBox(int status)
{
	bool is;//信号标志
	if (status==Qt::Checked)//如果被选中
		is = true;//置为true
	else
		is = false;//否则为false

	QString title = qobject_cast<QCheckBox*>(sender())->text();//强转为QString类型

	if (title == "File create")
	{
		emit event(1,is);//发送对应的信号

		QString tmp = KGlobalData::getGlobalDataIntance()->getEvents();//获取对应的变量
		tmp += "FC;";//设置事件
		KGlobalData::getGlobalDataIntance()->setEvents(tmp);//保存到全局

		return;
	}
	else if(title == "File delete")
	{
		emit event(2, is);//发送对应的信号

		QString tmp = KGlobalData::getGlobalDataIntance()->getEvents();//获取对应的变量
		tmp += "FD;";//设置事件
		KGlobalData::getGlobalDataIntance()->setEvents(tmp);//保存到全局

		return;
	}
	else if (title == "File change")
	{
		emit event(3, is);//发送对应的信号

		QString tmp = KGlobalData::getGlobalDataIntance()->getEvents();//获取对应的变量
		tmp += "FCH;";//设置事件
		KGlobalData::getGlobalDataIntance()->setEvents(tmp);//保存到全局

		return;
	}
	else if (title == "File rename")
	{
		emit event(4, is);//发送对应的信号

		QString tmp = KGlobalData::getGlobalDataIntance()->getEvents();//获取对应的变量
		tmp += "FR;";//设置事件
		KGlobalData::getGlobalDataIntance()->setEvents(tmp);//保存到全局


		return;
	}
	else if (title == "File remove")
	{
		emit event(5, is);//发送对应的信号

		QString tmp = KGlobalData::getGlobalDataIntance()->getEvents();//获取对应的变量
		tmp += "FM;";//设置事件
		KGlobalData::getGlobalDataIntance()->setEvents(tmp);//保存到全局
		
		return;
	}

	return;

}
```



#### 2.3数据库：kdatabase.h

1.初始化QSqlDatabase对象

```c++
void KDatabase::initDataBase()
{
    bool ok;
    //建立和SQlite数据库连接
    m_db = QSqlDatabase::addDatabase("QSQLITE");

    //设置数据库文件的名称
    m_db.setDatabaseName("database.db");

    //打开数据库
    if (m_db.open() == false)
    {
        qDebug() << "[KDatabase::initDataBase()]";
        return;
    }

    //判断表格名字
    if (!m_db.tables().contains("database"))
    {
        QSqlQuery query;
        QString sql = "CREATE TABLE database(id INTEGER PRIMARY KEY,"
            "day TEXT NOT NULL,"
            "time TEXT NOT NULL,"
            "operation TEXT NOT NULL,"
            "filename TEXT NOT NULL,"
            "suffix TEXT NOT NULL)";//创建表格

        ok = query.exec(sql);// 执行 SQL 语句
        if (!ok)
        {
            qDebug() << "[KDataBaseOperator::createTable]: ";
            return;
        }
    }
}
```

2.初始化QSqlTableModel对象

```c++
void KDatabase::initTableView()
{
    m_pModel = new QSqlTableModel(this);

    m_pModel->setTable("database");//数据库表与模型关联

    //设置手动提交模式，在修改模型后，不会自动更新到数据库
    m_pModel->setEditStrategy(QSqlTableModel::OnManualSubmit);

    m_pModel->select();//查询数据库，并将数据显示到关联的QTableView

    //设置表头
    m_pModel->setHeaderData(1, Qt::Horizontal, "day");
    m_pModel->setHeaderData(2, Qt::Horizontal, "time");
    m_pModel->setHeaderData(3, Qt::Horizontal, "operation");
    m_pModel->setHeaderData(4, Qt::Horizontal, "filename");
    m_pModel->setHeaderData(5, Qt::Horizontal, "suffix");
    
}
```

3.插入数据到数据库中

```c++
bool KDatabase::insertRecord(const QStringList record)
{
    bool ok;
    QSqlQuery query;
    // ? 占位符 可以使用 bindvalue 绑定变量,使用这个变量的值来替换占位符
    QString sql = "INSERT INTO database(id,day,time,operation,filename,suffix) VALUES(?,?,?,?,?,?); ";

    query.prepare(sql);

    //占位符绑定变量
    query.bindValue(0, record.at(0).toInt());//id
    query.bindValue(1, record.at(1));
    query.bindValue(2, record.at(2));
    query.bindValue(3, record.at(3));
    query.bindValue(4, record.at(4));
    query.bindValue(5, record.at(5));

    //执行 SQL 语句
    ok = query.exec();

    if (!ok)
    {
        emit addId();//发送ID增加信号
        qDebug() << "[KDataBaseOperator::insertRecord]: " ;
        return false;
    }

    return true;
}
```

4.多条信息遍历输出

```c++
bool KDatabase::searchRecord(const QString time)
{
    bool ok;
    QSqlQuery query;
    QString result;

    QString sql = "SELECT * FROM database WHERE day = ?";//创建sql语句

    query.prepare(sql);//预处理
    query.bindValue(0, time);//赋值

    ok = query.exec();
    if (!ok)
    {
        return false;
    }

    //遍历结果集合 单条记录 if() 多条记录 while()
    while (query.next())
    {
        //按照字段名从结果集合中获取对应的值
        result += query.value("time").toString();
        result += "\n";

        result += query.value("operation").toString();
        result += "\n";

        result += query.value("filename").toString();
        result += "\n";

    }

    KGlobalData::getGlobalDataIntance()->setResult(result);//显示

    emit printResult();//打印结果

    return true;
}
```

5.查找后缀

```c++
//查找后缀
void KDatabase::searchRecordSuffix(const QString suffix)
{
    //sql语句格式为QString sql = "suffix='txt' OR suffix='rtf'
    QString sql = "suffix=";
    QStringList subsuffix = KGlobalData::getGlobalDataIntance()->getSuffix().split(".");//.字符截断
    for (int i = 1; i < subsuffix.size(); ++i)
    {
        if (subsuffix.at(i) != " ")//按照结构处理sql
        {
            sql += "'";
            sql += subsuffix.at(i);
            sql += "'";
            sql += " ";
        }
        if (i != subsuffix.size() - 1)
            sql += "OR suffix =";
    }
    sql.trimmed();//去掉重复的空格

    m_pModel->setFilter(sql);//设置过滤
    m_pModel->select();//对比
    return;
}
```

#### 2.4关键字查询：kinquire.h

1.设置表头和关键字

```c++
//设置表头和关键字查询
void  KWatchFileTool::setTableHead()
{
    QString currentContent = m_pInquire->getChooseBox()->currentText();//获取表头
    QString indexContent = m_pInquire->getInquire()->toPlainText();//获取关键字
    KGlobalData::getGlobalDataIntance()->setTableHead(currentContent);//设置表头
    KGlobalData::getGlobalDataIntance()->setContent(indexContent);//设置关键字

    emit readyTableHead();
}

```

2.将信息保存到QStringList并且保存在全局函数中，等待调用

```c++
//组装QStringList并且保存在全局函数中，等待调用
void KShowDataWindow::combinationStringList()
{
    QStringList list = KGlobalData::getGlobalDataIntance()->getQstringList();//获取全局变量

    if (!list.isEmpty())//如果不为空
    {
        list.clear();//清除内容
        int id = KGlobalData::getGlobalDataIntance()->getId();
        ++id;//++id
        KGlobalData::getGlobalDataIntance()->setId(id);//设置序号
    }


    list.append(QString::number(KGlobalData::getGlobalDataIntance()->getId()));//插入序号
    list.append(KGlobalData::getGlobalDataIntance()->getDay());//插入日期
    list.append(KGlobalData::getGlobalDataIntance()->getTime());//插入时间
    list.append(KGlobalData::getGlobalDataIntance()->getOperation());//插入操作
    list.append(KGlobalData::getGlobalDataIntance()->getFileName());//插入文件名字
    list.append(KGlobalData::getGlobalDataIntance()->getSuffixName());//插入后缀

    KGlobalData::getGlobalDataIntance()->setQStringList(list);//设置list

    emit readyList();//发送准备好list信号

    return;
}

```



#### 2.5输出框图：kshowdatawindow.h

输出框图主要是输出监听到的被我们处理之后的信息

```c++
//新增文件操作
void KShowDataWindow::printfAddfile(QString fileName)
{
    QString addEvent;

    QDateTime current_date_time = QDateTime::currentDateTime();//获取本机时间
    QString current_date = current_date_time.toString("yyyy/MM/dd hh:mm:ss");//设置时间格式

    QString day = current_date_time.toString("yyyy/MM/dd");//获取日期
    KGlobalData::getGlobalDataIntance()->setDay(day);//保存日期到全局
    KGlobalData::getGlobalDataIntance()->setTime(current_date);//保存时间到全局

    current_date += ": ";
    QString username = KGlobalData::getGlobalDataIntance()->getUserName() + ":";//设置用户名
    current_date += username;

    addEvent = current_date + QStringLiteral("File create event,file name: ");//设置监听到的文件操作

    KGlobalData::getGlobalDataIntance()->setOperation("create");//保存操作到全局

    addEvent += KGlobalData::getGlobalDataIntance()->getFilePath();//设置文件路径
    addEvent += "/";
    addEvent += fileName;//设置文件名字

    if (!m_flag)
    {
        return;
    }
    m_pTextEdit->insertPlainText(addEvent);//显示结果
    m_pTextEdit->insertPlainText("\n");//换行

    QString tmp = KGlobalData::getGlobalDataIntance()->getStoreFileContent();//获取文件内容
    tmp += addEvent;
    tmp += "\n";
    KGlobalData::getGlobalDataIntance()->setStoreFileContent(tmp);//设置文件内容

    emit listenMassage();//发送信号

}
```

#### 2.6菜单栏：kmenubar.h

1.选择本地路径环境

```c++

//选择路径
void KWatchFileTool::selectFilePath()
{
    QFileDialog* fileDialog = new QFileDialog(this);//new 一个新的QFileDialog
    fileDialog->setFileMode(QFileDialog::Directory);//获取本地文件环境
    fileDialog->exec();//执行

    auto selectDir = fileDialog->selectedFiles();//提取出选取的路径
    KGlobalData::getGlobalDataIntance()->setStoreFilePath(selectDir);//保存到全局
    if (selectDir.size() > 0)
    {
        qDebug() << "Dir Path:" << selectDir.at(0);
    }

    emit readyPath();//发送信号

    delete fileDialog;//删除new对象
    fileDialog = nullptr;
}

```

2.保存数据

```c++
//保存数据
void KWatchFileTool::saveData()
{
    bool exist;
    QString fileName;
    QDir* folder = new QDir();
    QStringList tmp = KGlobalData::getGlobalDataIntance()->getStoreFilePath();//获取选择的路径
    QString filePath;
    for (int i = 0; i < tmp.size(); ++i)
    {
        filePath += tmp.at(i);
    }
    
    QByteArray data;
    data.append(KGlobalData::getGlobalDataIntance()->getStoreFileContent());//获取组装好的文件内容

    exist = folder->exists(filePath);//是否需要创建文件
    
    if (!exist)
    {
        bool ok = folder->mkdir(filePath);//创建文件
        if(ok)
            QMessageBox::warning(this, QStringLiteral("创建目录"), QStringLiteral("创建成功!"));
        else
            QMessageBox::warning(this, QStringLiteral("创建目录"), QStringLiteral("创建失败"));
    }
    
    filePath += "/%1.txt";//占位符设置名字
    fileName = filePath.arg(QStringLiteral("数据"));//设置为 数据.txt

    QFile file(fileName);
    if (!file.open(QIODevice::ReadWrite | QIODevice::Append | QIODevice::Text)) {//追加写入 添加结束符\r\n
        QMessageBox::warning(this, QStringLiteral("错误"), QStringLiteral("打开文件失败,数据保存失败"));
        return;
    }
    else {
        file.write(data);//写入数据
    }

    file.close();//关闭文件
    delete folder;//删除new对象
    folder = nullptr;
}

```



## 3.项目的漏洞与反思

**记录一下我写这份项目以来的漏洞是：**

- 在写主线程和子线程的过程中，我没有注意子线程的初始化函数和主线程初始化函数的分离，导致我在后来的漏洞检测中耗费了大量的力量去查找问题的源头。在前期的操作中，我一直认为出现泄漏的原因在于我未能在阻塞循环监听函数中退出，并且析构掉对象最后在讨论的过程中，我发现了问题的源头来自于，我在创建构造函数的时候，将子线程和父线程的构造函数放到了一起。
- 执行的时候，信号与槽函数connect（）可能会出现时间差，因此最好定好顺序再运行。

## 4.未来的改进方向

- 下次遇到多线程问题注意需要分清楚，子线程和主线程的构造函数分开，防止后面的出现的泄露位置。
- 信号与槽函数在使用的时候需要定好顺序，最好按照顺序创建。
- qdebug（）会解决大部分的问题。
- 数据库在处理的时候需要设置好序号，因为后期的设置可能会导致信息相同输入不进去，需要使用序号插入。