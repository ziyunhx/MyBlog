title: 在VS项目中通过GIT生成版本号作为编译版本号
date: 2013-11-01
categories: 
- 项目管理
tags:
- git
- VS
- 批处理
- 文件版本号

---

 上一篇博客写了如何在 .Net 项目使用 SVN 作为版本控制工具时生成与代码对应的组件版本号。虽然在公司一直使用 SVN ，但我却对 GIT 情有独钟，但少有文章提及如何具体在 Windows 平台来获得版本号。这让我有了迫切得到方法的希望。下面会具体实现如何在VS中使用Git版本号作为编译产生的文件版本号。

<!--more-->

 上篇博客 [《在VS项目中使用SVN版本号作为编译版本号》](https://www.tnidea.com/svn-project-version.html "在VS项目中使用SVN版本号作为编译版本号")

 将 GIT 的 commit 作为 .Net 项目编译后生成dll的文件版本号主要有以下几个困难。

 - GIT 没有一个数字的序号，而是一个SHA散列码；
 - GIT 提供的命令在 linux 十分方便，在 Windows 下需要额外的工具。

 第一个问题好解决，我们取当前文件夹 commit 的次数加上截取一段 SHA 码就可以作为文件版本号的最后一位。第二个问题目前想到的方法是调用 msysgit 提供的 Git Bash 来执行命令。

 好了，首先我们依旧得找到 msysgit 的安装目录，一查之下头就大了，各个地方的路径都感觉不靠谱，最后还是选用了
 `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1` 的 `InstallLocation` ，当然也可以通过 temp 环境变量来获取路径，安装时选择第二或第三项：

![git-setup](https://www.tnidea.com/media/image/git-setup.png)

 然后参照 Git Bash 的快捷方式拼接了下 call 的语句。写了一个 sh 文件来获得版本号，并保存到文件：

    # file name: git_ver.sh
    #!/bin/bash 
    VER_FILE=git_version.tmp
    LOCALVER=`git rev-list HEAD | wc -l | awk '{print $1}'`
    VER=r$LOCALVER
    VER="$VER $(git rev-list HEAD -n 1 | cut -c 1-7)"
    GIT_VERSION=$VER
    echo $GIT_VERSION>$VER_FILE

 在批处理里取出刚保存文件的值，接下来的工作就和使用 SVN 里的差不多了，唯一的区别是我们要自己实现关键字的替换。

 上篇博客 [《在VS项目中使用SVN版本号作为编译版本号》](https://www.tnidea.com/svn-project-version.html "在VS项目中使用SVN版本号作为编译版本号")

 建立以一个 AssemblyInfo.tpl 作为替换使用的模板，由于 AssemblyInfo.cs 中除了固定的值外还有类似 GUID 变化的值，所以我们不能全部替换，因此仅将需要修改的部分放在 tpl 中，内容如下：

    [assembly: AssemblyVersion("1.0.0.0")]
    [assembly: AssemblyFileVersion("1.0.0.GITVERSION")]

 自己替换 GITVERSION 字段为前面取到的版本号。

 接下来使用批处理替换原来的 AssemblyInfo.cs 文件，为了在每次编译时都自动替换，我们把调用批处理的命令卸载项目生成事件的生成前事件中：

    "(TargetDir)BeforeBuildProject.bat"(ProjectDir) $(TargetDir)

 批处理代码：

    ::-------------------------------------------------
    :: <sunmary>
    :: 根据指定工作目录信息和模板生成目标文件
    :: </sunmary>
    :: <param name="WorkDir">工作目录路径</param>
    :: <param name="Template">模板文件路径</param>
    :: <param name="target">生成目标文件的路径</param>
    :: <returns>执行结果</returns>
    ::=================================================
    
    ::-------------------------------------------------
    :: * Initialize *
    @ECHO OFF
    ::SetLocal EnableExtensions
    setlocal enabledelayedexpansion
    
    Rem Initialize Script arguments
    SET WorkDir=%1
    SET target=%2
    
    Rem Initialize Constants
    SET AssemblyInfo=ASSEMBLY_INFO.tmp
    
    Rem GoTo Main Entry
    GOTO Main
    ::=================================================
    
    ::-------------------------------------------------
    :: * Main Entry *
    :Main
    Rem Check arguments
    IF %WorkDir%=="" GOTO ARGUNENT_ERROR
    IF %target%=="" GOTO ARGUNENT_ERROR
    PushD %WorkDir%
    
    Rem Search TSVN Path
    
    For /f "tokens=1* delims=_" %%1 in ('reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1" /v "InstallLocation"^| findstr /i "InstallLocation"') Do (
      For /f "tokens=1*" %%3  in ("%%~2") Do (
    SET GIT_PATH=%%4
      )
    )
    
    COPY /y "%target%git_ver" "%WorkDir%git_ver"
    
    SET GIT_PATH="%GIT_PATH%bin\sh.exe" --login -i
    call %GIT_PATH% %WorkDir%git_ver
    
    for /f "delims=" %%i in (%WorkDir%\git_version.tmp) do (set VERSION=%%i)&(goto :next)
    :next
    DEL /Q "%WorkDir%\git_ver">NUL
    DEL /Q "%WorkDir%\git_version.tmp">NUL
    
    Rem Generate a template file
    
    COPY /y "%WorkDir%\Properties\AssemblyInfo.cs" "%WorkDir%\Properties\AssemblyInfo.cs.bak">NUL
    SET FILESTR="%WorkDir%\Properties\AssemblyInfo.cs"
    FindStr /v "AssemblyVersion AssemblyFileVersion" %FILESTR%>%AssemblyInfo%
    
    For /f "delims=" %%k In (%target%\AssemblyInfo.tpl) do (
      set str=%%k
      set str=!str:GITVERSION=%VERSION%!
      echo !str! >> "%AssemblyInfo%"
    )
    
    COPY /y "%AssemblyInfo%" "%WorkDir%\Properties\AssemblyInfo.cs"
    GOTO SUCCESS
    ::=================================================
    
    ::-------------------------------------------------
    :: * Error Handlers *
    :ARGUNENT_ERROR
    ECHO 传入的参数无效。
    GOTO FAIL
    
    :UNKNOE_ERROR
    ECHO 未知错误。
    GOTO FAIL
    ::=================================================
    
    ::-------------------------------------------------
    :: * Program Exit *
    :FAIL
    DEL /Q "%AssemblyInfo%">NUL
    ::IF EXIST "%WorkDir%Properties\AssemblyInfo.cs.bak" (COPY /y "%WorkDir%Properties\AssemblyInfo.cs.bak" "%WorkDir%Properties\AssemblyInfo.cs"&DEL /q "%WorkDir%Properties\AssemblyInfo.cs.bak")>NUL
    ECHO "error"
    popd
    EXIT 1
    
    :SUCCESS
    DEL /Q "%AssemblyInfo%">NUL
    ECHO "success"
    popd
    EXIT 0
    ::=================================================
	

 惯例附上所有代码：[点击下载](http://pan.baidu.com/s/1osg9C "GIT生成版本号.rar")