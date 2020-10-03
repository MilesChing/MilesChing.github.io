---
title:  Guide to Thunder X
date:   2019-07-25
tags: [Thunder X,公开文档]
cover: /posts/thunder_x/images/thunder-cover.jpg
---
Welcome to the guide to Thunder X! Visit [Windows store](https://www.microsoft.com/en-us/p/thunder-x/9njqw2wdtd43?activetab=pivot:overviewtab) to download our appication in case you don't know much about Thunder X. For a copy of our source code, view [our github page](https://github.com/MilesChing/ThunderX).

<!--more-->

- [简体中文版本](https://milesching.github.io/thunder-x/2019/06/08/ThunderX_zh_cn.html)

## Update List

### Version 1.9.6.0

- Solve the path error caused by modifying the default folder when a new task page activated;
- Resolve exceptions that may result from multiple tasks finishing at the same time;
- Speed optimization and interface adjustment;

## Helpful Tips

### Main Page

The main page is the first page you see each time you open your app. It is recommended that you use Thunder X at a narrow-width mode (except for the built-in browser), as shown in the screenshot below. We display histories of completed tasks on the main screen respectively in version `1.9.0.0`. We provide three buttons: open file, open folder, and clear record for you to find tasks you downloaded earlier,

> Tip: your file will not be removed along with it's record.

![Main Page](../images/ThunderX帮助页截图_主屏幕.png)

Since the intent of the Compact Overlay view mode is a very specific type of user experience
We implemented a Compact Overlay view mode for the main page. You can easily switch between view modes by clicking the button in the lower left corner. The Compact Overlay view mode means your page will be displayed in a small-sized window which is always on the top. You can also manipulate the downloader in this mode, absolutely.

![Compact Overlay](../images/ThunderX帮助页截图_画中画.png)

### New Task Page

The experience of creating a new task is simple and easy. All you need is to fill the input box and wait for the app to automatically analyze the feasibility and give a hint. Thunder X supports the following types of links:

- Links based on HTTP/HTTPS
  - Links to YouTube playpages
  - Other web links
- Links based on Thunderbolt protocol

> Note: some links encapsulate magnetic chains using the Thunderbolt protocol. Thunder X can parse these kind of links but does not support for downloading.

Under normal circumstances, the software will automatically resolve links. For most links, you will see a slider to choose the number of threads, which will be set as the default number when you are not going to change it.

> Tip: it's not always better to choose more threads. The best plan depends on your network, size of the file, and your PC's current resource usage. Use less than five threads for smaller files for best results, and more for larger files.

![New Task Page](../images/ThunderX帮助页截图_新增任务.jpg)

You are allowed to rename the downloaded file in all cases. If you choose not to do so we will use the name inferred from the link and the response from the server as the file name. In version '1.9.0.0' we updated the extension speculation algorithm for files. After the program "presumes" to infer the extension, it will also give you several recommended extensions as an alternative. If you know the actual type of file you are going to download you can also choose these extensions.

### Automaticlly Reconnect

Most of errors can be resolved by reconnecting. This feature has been built in to avoid manual operations. Of course, you need to set an upper limit for this behavior. When the number of errors reaches your upper limit (default is 50, you can adjust this value in Settings), we will notify you of the last error and stop automatically reconnecting. You can still manually retry after the error number reaches the upper limit. Each time you reconnect manually, the current error number will reset to 0.

### Built-in Browser

The browser is built in because download links to many resources do not hang directly on the web page. Most of them are generated dynamically by the remote server. We can automatically analyze download links that you may need from the page using a buit-in browser. All you need is to visit the site like you are using a real browser.

Let's look at [the download page of Ubuntu](https://www.ubuntu.com/download/desktop)：

![Download Page of Ubuntu](../images/ThunderX帮助页截图_Ubuntu下载页面.jpg)

Click on the download button and you will be navigated to a page which will thank you for downloading the resource. Just click the menu button to open the sidebar (you may have to wait a few seconds to let us connect to the server) and select a link you want. Click on our button and you will be automatically navigated to the new task page.

![Usage of Link List](../images/ThunderX帮助页截图_链接列表.jpg)

We will continue to optimize the experience of the built-in browser (mainly in the area of link analysis). If you have any good idea, feel free to send it to us.

### Complete the Task

After the download task is completed, we will automatically move the file to your download folder. We will save the file somewhere inside the software and let you know this when your download folder cannot be visited. You can click "access folder" in the notification to get to the true folder or open the file in a download history.

## Other Matters

### Differences Between the Trial Version and the Full Version

The trial version locks multi-threaded modules. Which means that users can only use a single thread to download when running a trial version. Other functions are not affected at all.

### About Thunder X

Thunder X has no connection with Xunlei Limited, and the software does not use the company's technology or achievements.

Email: `qinzhengrui@live.cn`

Weibo: `@MilesChing`
