title: 在VS项目中使用SVN版本号作为编译版本号
date: 2013-10-27
categories: 
- 项目管理
tags: 
- SVN
- VS
- 项目管理
- 批处理
- 文件版本号

---

 在生产项目中，版本号是必不可少的一部分。版本号的规则也有许多种，在此不讨论具体的编码规范。对于迭代的产品，版本繁多，特别是有多个实施项目所使用产品的版本不同（基于定制需求）时，清楚的标识组件与代码的对应关系十分重要。 本文主要说明如何在 .Net 项目使用 SVN 作为版本控制工具时生成与代码对应的组件版本号。

<!--more-->

 我们知道，SVN 在 commit 时会生成一串数字作为序号，所以基本思路是把这个序号作为 .Net 项目编译后生成dll的文件版本号的最后一段。下面所列方法需要使用到TortoiseSVN 提供的 SubWCRev.exe 。

 首先，我们需要通过注册表查找 TortoiseSVN 的安装目录。

    Rem Search TSVN Path
    For /f "tokens=*" %%i In ('Reg Query HKLM\Software\TortoiseSVN /v Directory') Do (
       ECHO %%i | Find "Directory">NUL
       IF %ERRORLEVEL% == 0 (For /f "tokens=1,2,*" %%j In ("%%i") Do (SET TSVN_PATH=%%1))
    )
    SET TSVN_PATH=%TSVN_PATH%bin\SubWCRev.exe


 SubWCRev 是通过替换文件中指定的关键字来实现获得 commit 序号的，点击查看详细的列表。 

 然后我们建立以一个 AssemblyInfo.tpl 作为替换使用的模板，由于 AssemblyInfo.cs 中除了固定的值外还有类似 GUID 变化的值，所以我们不能全部替换，因此仅将需要修改的部分放在 tpl 中，内容如下：

    [assembly: AssemblyVersion("1.0.0.0")]
    [assembly: AssemblyFileVersion("1.0.0.$WCREV$")]

 接下来使用批处理替换原来的 AssemblyInfo.cs 文件，为了在每次编译时都自动替换，我们把调用批处理的命令写在项目生成事件的生成前事件中，例如下面这样：

    "(TargetDir)BeforeBuildProject.bat""(ProjectDir)" "$(TargetDir)AssemblyInfo.tpl" .\Properties\AssemblyInfo.cs

 `$(TargetDir)`表示编译输出目录，更多可用全局变量请在生成事件中点击“宏”查看。

 替换 AssemblyInfo.cs 的批处理代码：


    SET WorkDir=%1
    SET Template=%2
    SET target=%3
    SET AssemblyInfo=ASSEMBLY_INFO.tmp
    
    PushD %WorkDir%
    SET WorkDir=.\
    
    Rem Generate a template file
    FindStr /v "AssemblyVersion AssemblyFileVersion" %target% > %AssemblyInfo%
    FindStr ".*" %Template% >> %AssemblyInfo%
    
    Rem Using TSVN Replace Tlp
    "%TSVN_PATH%" %WorkDir% %AssemblyInfo% %target%>NUL

 当然这样还不是一劳永逸，你会发现每次编译 AssemblyInfo.cs 文件都会变化，因此你的 commit 序号也会一直跟着增加，这并不是我们所想要的效果。这里提出一种解决方案，在每次替换后生成 dll 后又将 AssemblyInfo.cs 还原回去。

 为此，我们在生成前事件中增加备份命令：

    COPY /y "%target%" "%target%.bak">NUL

 然后增加生成后事件，调用命令为：

    "(TargetDir)AfterBuildProject.bat""(ProjectDir)Properties\AssemblyInfo.cs"

 在 AfterBuildProject.bat 中我们需要完成还原  AssemblyInfo.cs 和删除备份文件的工作，代码如下：

    SET target=%1
    COPY "%target%.bak" %target%
    DEL /Q "%target%.bak" 2>NUL

 就此，每次编译时，系统都会自动将 commit 序号放到 dll 的文件版本中了。

 完整代码下载：[点击下载](http://pan.baidu.com/s/1pD5TJ "SVN生成版本号.rar")