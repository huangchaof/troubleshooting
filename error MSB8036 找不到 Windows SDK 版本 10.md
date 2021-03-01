 `error MSB8036: 找不到 Windows SDK 版本 10.0.18362.0。请安装所需版本的 Windows SDK`

`error MSB8036 The Windows SDK version 10.0.18362.0 was not found`



## 错误调查

我在E盘上安装了vs2019 community，并安装了WindowsSdk version `10.0.17763.0和10.0.18362.0`，创建控制台工程后遇到了上面的错误，而且还发现了其他问题。**~~既然已经安装了10.0.18362.0，就不应该出现找不到信息~~**

**我的win10版本是1909（18363.1379**）

下面这些问题也是这个错误的表现

1. vs2019创建控制台工程默认使用windowssdk最新版本

2. 由于本机上安装的最新版本是10.0.18362.0，build时会有如下提示：

   中文提示：

```
错误	MSB8036	找不到 Windows SDK 版本 10.0.18362.0。请安装所需版本的 Windows SDK，或者在项目属性页中或通过右键单击解决方案并选择“重定解决方案目标”来更改 SDK 版本。	Project1	E:\Microsoft Visual Studio\2019\MSBuild\Microsoft\VC\v160\Microsoft.Cpp.WindowsSDK.targets
```

​	英文提示：

```
Error	MSB8036	The Windows SDK version 10.0.18362.0 was not found. Install the required version of Windows SDK or change the SDK version in the project property pages or by right-clicking the solution and selecting "Retarget solution".	ConsoleApplication10	E:\Microsoft Visual Studio\2019\MSBuild\Microsoft\VC\v160\Microsoft.Cpp.WindowsSDK.targets	46	
```

查看文件`Microsoft.Cpp.WindowsSDK.targets`会发现变量`WindowsSDKInstalled`为false会导致上面的错误信息.在project的属性页切换Windows SDK为10.0.17763.0，build成功

观察到`$(WindowsSdkDir)`的值在不同版本的sdk下，值不同

![](https://gitee.com/lensousou/imagerepo/raw/master/img//20210301232651.jpg)

切换为10.0.17763.0版本后，路径正确了

![image-20210302011030477](https://gitee.com/lensousou/imagerepo/raw/master/img//20210302011030.png)

3.安装boost`vcpkg install boost` ,当安装到包python3[core]:x86-windows -> 3.9.2会调用msbuild,运行的命令：

```
Command failed: msbuild E:/Vcpkg/vcpkg/buildtrees/python3/x86-windows-rel/v3.9.2-cc9ebdcfd9.clean/PCbuild/pcbuild.proj /p:Configuration=Release /p:IncludeExtensions=true /p:IncludeExternals=true /p:IncludeCTypes=true /p:IncludeSSL=true /p:IncludeTkinter=false /p:IncludeTests=false /p:ForceImportBeforeCppTargets=E:/Vcpkg/vcpkg/buildtrees/python3/src/v3.9.2-cc9ebdcfd9.clean/PCbuild/python_vcpkg.props /p:IncludeUwp=false /p:_VcpkgPythonLinkage=DynamicLibrary /t:Rebuild /p:Platform=Win32 /p:PlatformToolset=v142 /p:VCPkgLocalAppDataDisabled=true /p:UseIntelMKL=No /p:WindowsTargetPlatformVersion=10.0.18362.0 /p:VcpkgTriplet=x86-windows /p:VcpkgInstalledDir=E:/Vcpkg/vcpkg/installed /p:VcpkgManifestInstall=false /m
```

也会产生找不到windowssdk的错误：

```
E:\Microsoft Visual Studio\2019\MSBuild\Microsoft\VC\v160\Microsoft.Cpp.WindowsSDK.targets(46,5): error MSB8036: 找不到 Windows SDK 版本 10.0.18362.0。请安装所需版本的 Windows SDK，或者在项目属性页中或通过右键单击解决方案并选择“重定解决方案目标”来更改 SDK 版本。 
```

 msbuild使用proj文件构建，使用的sdk版本也是10.0.18362.0

```
msbuild pcbuild.proj  /p:WindowsTargetPlatformVersion=10.0.18362.0
```

注意msbuild可以用/p:WindowsTargetPlatformVersion=10.0.18362.0形式制定sdk版本

 4.运行cmake,generator使用vs2019 2016,也会报错： error MSB8036: 找不到 Windows SDK 版本 10.0.18362.0，请安装所需版本的 Windows SDK

![image-20210301233034660](https://gitee.com/lensousou/imagerepo/raw/master/img//20210301233034.png)

## 何处下载sdk

1. 从vs installer处单个组件下载安装

2. 各个版本的Windows SDK 下载

   https://developer.microsoft.com/zh-cn/windows/downloads/sdk-archive

   在此处并没有找到10.0.18362.0，10.0.18362.0是随着vs2019 community安装的
   
   经安装，发现下面的10.0.18362.1就是10.0.18362.0，用vs installer卸载了10.0.18362.0，安装此处的10.0.18362.1，还是报相同错误
   
   10.0.18362.1
   
   ![image-20210302003631864](https://gitee.com/lensousou/imagerepo/raw/master/img//20210302003639.png)
   
   此处似乎有网络问题，需要先下载iso文件

3.最新版本的Windows SDK下载

​	https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk

### 卸载sdk

用windows卸载功能

![image-20210302004838307](https://gitee.com/lensousou/imagerepo/raw/master/img//20210302004838.png)

## 尝试解决

注意到工程的proj文件里用`WindowsTargetPlatformVersion`控制sdk版本，在vs安装目录下查找WindowsTargetPlatformVersion发现2019\Common7\IDE\VC\VCWizards\2052\common.js里有个函数
`StampWindowsTargetPlatformVersion`设置了它的值

```
function StampWindowsTargetPlatformVersion(oProj)
{
    var strLatestWindowsSDKVersion = GetLatestWindowsSDKVersion(oProj);
    if (strLatestWindowsSDKVersion)
    {
        //Stamp the latest Windows SDK version found in the disk - It is a global property, so enough to do it for any one config
        var commonConfigRule = oProj.Object.Configurations(1).Rules("ConfigurationGeneral");
        commonConfigRule.SetPropertyValue("WindowsTargetPlatformVersion", strLatestWindowsSDKVersion);
    }
}
```

修改了sdk版本为某个特点值，还是不能解决上面的问题



用vs installer修复也没有解决



## 最终解决

我一直在vs installer里修复，或者尝试修改vs的2019\Common7\IDE\VC\VCWizards\2052\common.js，或者在网上搜索，寻找解决办法，都没有解决。

在我安装了一个新sdk，由于无法通过vs installer卸载（因为不是通过installer安装的），尝试用windows自带的卸载功能时，发现可以在此处修复sdk.使用修复后，这个错误消失了。

![image-20210302011611271](https://gitee.com/lensousou/imagerepo/raw/master/img//20210302011611.png)










