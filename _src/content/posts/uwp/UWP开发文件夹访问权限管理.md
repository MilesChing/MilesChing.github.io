---
title:  UWP开发：文件和文件夹权限管理
date:   2019-08-07
tags: [UWP]
---

在做UWP开发的时候，为用户保存一个至多个默认文件夹作为本地存储空间是我们经常遇到的需求。文件夹的访问和管理不当容易导致拒绝访问的异常，增加用户的时间成本。本文主要讨论如何在UWP应用中对于文件和文件夹的实例获取，以及如何申请和管理对于文件和文件夹的访问权限，对于该权限机制的操作逻辑作简要记录。

<!--more-->

## 使用`StorageFile`类和`StorageFolder`类实例操纵文件和文件夹

### 从路径和`Uri`获取实例

涉及文件和文件夹操作时，我们最熟悉不过的就是`StorageFile`和`StorageFolder`，它们的实例与某一文件或文件夹相对应，许多与文件信息提取和文件操作有关的方法都直接依赖于这两个类的实例。

你可以从地址或`Uri`实例取得`StorageFile`实例：

~~~csharp
string path = "your path";
StorageFile file = await StorageFile.GetFileFromPathAsync(path);
file = await StorageFile.GetFileFromApplicationUriAsync(new Uri(path));
~~~

这里使用`Uri`的区别在于后者可以利用协议（`Uri`前缀）访问一些特殊位置，这里引用另一篇博客的内容作为说明，更多使用方法可参考[微软官方的文档](https://docs.microsoft.com/en-us/previous-versions/windows/apps/hh965322(v=win.10))，

> 应用在安装了之后，会自动生成三个文件夹存放应用的数据，分别是Local文件夹（用来存放本地数据）、Roaming文件夹（用来存放漫游数据）和Temp文件夹（用来寻访临时数据）。在程序中访问这三个文件夹中的文件，需要使用的 `Uri`的前缀应该是`ms-appdate://`，后面加上路径，比如`/local/a.mp3`，需要注意的是路径最前面的`/`代表了路径是从根目录开始的绝对路径，因此完整的URI定义应该是`new Uri("ms-appdata:///local/a.mp3")`；
>
> 如果需要访问应用程序包中的资源，也就是在开发中显示在解决资源管理器中的资源，比如`Assets`中的资源，需要使用的前缀是`ms-appx://`，后面的路径与上一段讲的一样，比如访问`Assets`中的一个`mp3`文件使用的URI就是`ms-appx:///Assets/a.mp3`。

同样地使用异步方法获取`StorageFolder`实例：

~~~csharp
StorageFolder folder = await StorageFolder.GetFolderFromPathAsync(path);
~~~

另外，借助`StorageFolder`实例，你也可以获取文件夹中的子文件和子文件夹：

![子文件和子文件夹](../images/文件和文件夹权限_子文件和子文件夹.png)

还有众多移动、复制、删除、在根目录下创建新子文件和新子文件夹等等操作，以及使用`FileIO`类结合`StorageFile`实例进行文件读写的操作，都非常基本。这里不做详细介绍。

### 用户选择的文件和文件夹

使用`Windows.Storage.Pickers.FolderPicker`可以管理一个文件夹选取器，该选取器可以弹出一个我们常见到的选取文件夹窗口供用户选择，用户选择后异步函数将直接返回一个`StorageFolder`实例。感谢C#的异步机制这个过程的实现相当优雅：

~~~csharp
FolderPicker picker = new FolderPicker();
picker.FileTypeFilter.Add("."); //虽然是选取文件夹，FileTypeFilter也不能为空，这里随意添加一项
StorageFolder folder = await picker.PickSingleFolderAsync();
~~~

我们经常看到的文件夹选取窗口就来自这几行代码：

![文件夹选取窗口](../images/文件和文件夹权限_文件夹选取器.png)

类似地，你可以用`FileOpenPicker`选取文件：

~~~csharp
FileOpenPicker picker = new FileOpenPicker();
picker.FileTypeFilter.Add(".txt");
StorageFile file = await picker.PickSingleFileAsync();
IReadOnlyList<StorageFile> files = await picker.PickMultipleFilesAsync();
~~~

或使用`FileSavePicker`选取用户选择保存到某处后创建的新文件：

~~~csharp
FileSavePicker picker = new FileSavePicker();
picker.FileTypeChoices.Add(".png");
StorageFile file = await picker.PickSaveFileAsync();
~~~

### 从`StorageItemAccessList`获取

这里就要涉及到权限管理的部分了，我们在下一小节中作具体介绍。

## 管理文件和文件夹访问权限

首先，我们对以上三种获取`StorageFile`和`StorageFolder`实例的方法作以总结。总体上，我们获取一个已存在的文件或文件夹只有这三种途径：

1. 从路径和`Uri`获取实例；
2. 获取用户选择的文件和文件夹；
3. 从`StorageItemAccessList`获取文件和文件夹；

文件访问权限是什么？**系统维护了一个列表，它把我们运行时可能会进行访问、修改、移动、写入等等操作的文件记录下来，这样在你操作的时候，它会查看你提供的目标文件或文件夹是否在他的列表里，如果不在，那么你没有权限进行读取或修改**，因此所有这些操作都需要你的操作对象在这个列表中，否则相应的函数就会抛出异常。

当然，在你的权限列表空无一物时，你也有一些位置是默认可以访问的，如我们常用的`ApplicationData.Current.LocalCacheFolder`是一个本地缓存文件夹，你可以在里面存储一些备份或配置文件等等。要了解更多这些位置可以查看[file-access-permissions](https://docs.microsoft.com/zh-cn/windows/uwp/files/file-access-permissions)。

### 权限系统的运作规则

那么首先我们要知道的是，**当你获取到`StorageFolder`或`StorageFile`实例时，你就已经能够对该文件夹或文件进行操作，但是这不代表你可以长久地对它进行操作**。由于方法2是用户手动选择了文件或文件夹，相当于是默许应用程序使用它，因此不需要访问权限也能获取到实例。但`StorageFile`和`StorageFolder`不可持久化，即软件关闭后你就再也不能获得该文件，而你又不想让用户每次都手动选择文件夹。因此，系统提供给我们一个结构，可以将我们将要访问的文件和文件夹存储在里面，并且即使我们的软件关闭后重新启动也能访问到这些文件和文件夹。

这个结构指的就是`StorageItemAccessList`类。我们要用到的实例有两个，分别是`StorageApplicationPermissions`类的两个静态属性：`MostRecentlyUsedList`和`FutureAccessList`。它们二者均可存储文件和文件夹，`MostRecentlyUsedList`代表了当前要访问的文件和文件夹，存储上限为25项，而`FutureAcceessList`代表未来即将访问的文件和文件夹，存储上限为1000项。实际上二者有相同的功效，文件或文件夹在其中任意一个列表中都会具有访问权限，对于两个列表中的具体分工，[UWP开发文档](https://docs.microsoft.com/zh-cn/windows/uwp/files/how-to-track-recently-used-files-and-folders)中有以下描述：

> 除了`MostRecentlyUsedList`，你的应用还具有一个未来访问列表。通过选择文件和文件夹，你的用户向你的应用授予访问项的权限（若未授权可能无法访问）。如果将这些项添加到你的未来访问列表，然后当你的应用需要以后重新访问这些项时将保留该权限。

### 跟踪最近使用的文件和文件夹

最初，在应用中持久化地（持久化地，即能存储在文件中的，转化为基本数据类型的存储方式）存储文件和文件夹的方式基本为使用路径字符串存储。实际上，`StorageItemAccessList`为我们提供了更好的选择：当你将某文件或文件夹存储在一个`StorageItemAccessList`中时，`Add`方法会返回一个`token`字符串，该字符串与每个文件或文件夹一一对应，也就是当你重复添加文件到列表时，得到的`token`一定相同。可以看出，`StorageItemAccessList`实际上并非一个列表，而是一个以键值对为基础的字典，我们对`StorageItemAccessList`中的项进行操作都依赖`token`作为索引。如下代码以`FutureAccessList`为例展示了一些对`StorageItemAccessList`实例的基本操作。

~~~csharp
StorageFolder folder = await StorageFolder.GetFolderFromPathAsync("test");
StorageFile file = await StorageFile.GetFileFromPathAsync("test.t");

var FAList = Windows.Storage.AccessCache.StorageApplicationPermissions.FutureAccessList;

string fileToken = FAList.Add(file);    //添加文件
string folderToken = FAList.Add(folder);    //添加文件夹

bool access = FAList.CheckAccess(folder);   //查看是否有访问权限
bool contains = FAList.ContainsItem(folderToken);   //查看是否有某token

file = await FAList.GetFileAsync(fileToken);    //获取文件
folder = await FAList.GetFolderAsync(folderToken);  //获取文件夹

FAList.Remove(fileToken); //删除文件
FAList.Remove(folderToken);   //删除文件夹
~~~

> `StorageItemAccessList`似乎在项过多时新加入的项会自动顶出最早加入的项。因此有必要注意其存储上限，并时刻注意检查权限。

除此之外，若要遍历当前添加的项，可以使用`StorageItemAccessList.Entries`，它是一个`AccessListEntryView`实例：

~~~csharp
public sealed class AccessListEntryView : IReadOnlyList<AccessListEntry>, IEnumerable<AccessListEntry>
~~~

其中`AccessListEntry`包含了特定元数据和对应的`token`，以防万一你有遍历所有的`token`的需求。

更多使用方法可以查阅[`StorageItemAccessList`类的文档](https://docs.microsoft.com/en-us/uwp/api/Windows.Storage.AccessCache.StorageApplicationPermissions)。
