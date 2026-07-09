# Windows 从零编译打包 Python 3.11.15 独立单文件安装包完整手册

## 文档说明

本文为踩坑后标准化最优流程，全程使用官方 `PCbuild` + `Tools/msi` WiX 打包链路，**规避绿色编译文件夹迁移崩溃、MSBuild v143 工具集缺失、多平台文件缺失、安装包多文件依赖**四大核心问题；最终输出**单个 exe 独立安装包**，拷贝到任意新 Windows 机器即可完整安装，内置解释器、标准库、tkinter、pip、右键文件关联。

## 一、前置环境准备

### 1. 必备软件

1. Visual Studio 2022（安装组件：**MSVC v143 x64/x86 生成工具、Windows SDK 10+**）
2. Git（拉取 CPython 源码）
3. WiX 3.14（源码 externals 自动集成，无需单独安装）
4. 管理员权限终端

### 2. 源码获取

cmd







```
git clone https://github.com/python/cpython.git
cd cpython
git checkout v3.11.15
```

### 3. 拉取外部依赖（tcltk、WiX、VC redist）

cmd







```
cd PCbuild
get_externals.bat
```

执行完成后 `externals` 目录会生成 `tcltk`、`windows-installer` 打包依赖。

### 4. 固定终端（全程必须使用）

开始菜单打开：`x64 Native Tools Command Prompt for VS 2022`

> 关键：此终端完整继承 VC143 编译环境，避免 WiX 独立 MSBuild 进程报工具集缺失。

## 二、预编译 64 位 Shell 扩展 pyshellext.dll（解决打包阻塞根源）

官方打包脚本会自动并行编译 32/64/ARM64 三平台 shell 扩展，独立 MSBuild 进程丢失 VS 环境直接报错；

我们仅需要 64 位包，提前手动编译 64 位 dll，后续裁剪 32/ARM64 相关代码：

cmd







```
cd PCbuild
msbuild pyshellext.vcxproj /p:Platform=x64;Configuration=Release
```

编译成功标志：输出文件 `PCbuild\amd64\pyshellext.dll`。

## 三、修改 WiX 打包工程（两处核心修改，消除编译报错）

### 修改 1：`Tools\msi\launcher\launcher.wixproj`

删除自动编译 32/64/ARM64 pyshellext 的三段 Target，完整替换文件内容：

xml







```
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <PropertyGroup>
        <ProjectGuid>{921CF0E6-AEBC-4376-BA1D-CD46EBFE6DA5}</ProjectGuid>
        <SchemaVersion>2.0</SchemaVersion>
        <OutputName>launcher</OutputName>
        <OutputType>Package</OutputType>
        <DefineConstants>UpgradeCode=1B68A0EC-4DD3-5134-840E-73854B0863F1;SuppressUpgradeTable=1;$(DefineConstants)</DefineConstants>
        <IgnoreCommonWxlTemplates>true</IgnoreCommonWxlTemplates>
        <SuppressICEs>ICE80</SuppressICEs>
        <_Rebuild>Build</_Rebuild>
    </PropertyGroup>
    <Import Project="..\msi.props" />
    <ItemGroup>
        <Compile Include="launcher.wxs" />
        <Compile Include="launcher_files.wxs" />
        <Compile Include="launcher_reg.wxs" />
    </ItemGroup>
    <ItemGroup>
        <EmbeddedResource Include="*.wxl" />
    </ItemGroup>

    <Target Name="_MarkAsRebuild" BeforeTargets="BeforeRebuild">
      <PropertyGroup>
        <_Rebuild>Rebuild</_Rebuild>
      </PropertyGroup>
    </Target>

    <!-- 仅保留py/pyw启动器编译，删除全部pyshellext自动编译Target -->
    <Target Name="_EnsurePyEx86" Condition="!Exists('$(BuildPath32)py.exe') or '$(_Rebuild)' == 'Rebuild'" BeforeTargets="PrepareForBuild">
        <MSBuild Projects="$(PySourcePath)PCbuild\pylauncher.vcxproj" Properties="Platform=Win32" Targets="$(_Rebuild)" />
    </Target>
    <Target Name="_EnsurePywEx86" Condition="!Exists('$(BuildPath32)pyw.exe') or '$(_Rebuild)' == 'Rebuild'" BeforeTargets="PrepareForBuild">
        <MSBuild Projects="$(PySourcePath)PCbuild\pywlauncher.vcxproj" Properties="Platform=Win32" Targets="$(_Rebuild)" />
    </Target>
    
    <Import Project="..\msi.targets" />
</Project>
```

### 修改 2：`Tools\msi\launcher\launcher_files.wxs`

删除 32 位、ARM64 pyshellext 打包节点，仅保留 amd64 版本，消除 `LGHT0103 文件不存在` 链接错误：

xml







```
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
    <Fragment>
        <ComponentGroup Id="launcher_exe">
            <Component Id="py.exe" Directory="LauncherInstallDirectory" Guid="{B5107402-6958-461B-8B0A-4037D3327160}">
                <File Id="py.exe" Name="py.exe" Source="py.exe" KeyPath="yes" />
                <RegistryValue Root="HKMU" Key="Software\Python\PyLauncher" Value="[#py.exe]" Type="string" />
            </Component>
            <Component Id="pyw.exe" Directory="LauncherInstallDirectory" Guid="{8E52B8CD-48BB-4D74-84CD-6238BCD11F20}">
                <File Id="pyw.exe" Name="pyw.exe" Source="pyw.exe" KeyPath="yes" />
            </Component>

            <Component Id="launcher_path_cu" Directory="LauncherInstallDirectory" Guid="{95AEB930-367C-475C-A17E-A89BFCD4C670}">
                <Condition>NOT ALLUSERS=1</Condition>
                <RegistryValue KeyPath="yes" Root="HKMU" Key="Software\Python\PyLauncher" Name="InstallDir" Value="[LauncherInstallDirectory]" Type="string" />
                <Environment Id="PATH_CU" Action="set" Name="PATH" Part="first" Value="[LauncherInstallDirectory]" />
            </Component>
            <Component Id="launcher_path_lm" Directory="LauncherInstallDirectory" Guid="{4A41C365-4E27-4D38-A6D1-4A01B4A6500C}">
                <Condition>ALLUSERS=1</Condition>
                <RegistryValue KeyPath="yes" Root="HKMU" Key="Software\Python\PyLauncher" Name="InstallDir" Value="[LauncherInstallDirectory]" Type="string" />
            </Component>

            <!-- 仅保留64位shell扩展，移除win32/arm64组件 -->
            <Component Id="pyshellext_amd64.dll" Directory="LauncherInstallDirectory" Guid="{E7411EFD-F1DD-40EB-B0C7-4BA02BF3E75F}" Win64="yes">
                <Condition>VersionNT64 AND NOT ARM64_SHELLEXT</Condition>
                <File Id="pyshellext_amd64.dll" Name="pyshellext.amd64.dll" Source="!(bindpath.Build64)\pyshellext.dll">
                    <Class Id="{BEA218D2-6950-497B-9434-61683EC065FE}" Advertise="no" Context="InprocServer32" ThreadingModel="apartment" />
                </File>
            </Component>
        </ComponentGroup>
    </Fragment>
</Wix>
```

## 四、修改 Bundle 打包配置：开启单 EXE 内嵌全部 MSI（关键）

原生打包输出的 exe 仅 1MB，仅为引导器，依赖同目录一堆 msi；修改后所有组件内嵌 exe，实现单文件分发。

### 修改 1：`Tools\msi\bundle\snapshot.wixproj`

新增 `<EmbedPayloads>true</EmbedPayloads>`，完整文件：

xml







```
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <PropertyGroup>
        <ProjectGuid>{8A4A1162-4BF9-4FF6-9A98-315F01E44932}</ProjectGuid>
        <OutputName>python</OutputName>
        <OutputSuffix></OutputSuffix>
        <!-- 开启内嵌全部MSI负载，实现单exe -->
        <EmbedPayloads>true</EmbedPayloads>
    </PropertyGroup>

    <Import Project="..\msi.props" />

    <PropertyGroup>
        <DefineConstants Condition="'$(Pack)' != 'true'">
            $(DefineConstants);CompressMSI=no;
        </DefineConstants>
        <DefineConstants Condition="'$(Pack)' == 'true'">
            $(DefineConstants);CompressMSI=yes;
        </DefineConstants>
        <DefineConstants>
            $(DefineConstants);
            CompressPDB=no;
            CompressMSI_D=no;
        </DefineConstants>
    </PropertyGroup>
    
    <Import Project="bundle.targets" />
</Project>
```

### 修改 2：`Tools\msi\bundle\snapshot.wxs`

所有 `<MsiPackage>` 节点增加 `EmbedCab="yes"`，将 MSI 打包进 exe 内部，示例标准写法：

xml







```
<Chain>
  <MsiPackage SourceFile="$(BuildPath)\ucrt.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\core.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\lib.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\pip.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\launcher.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\tcltk.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\doc.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\dev.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\test.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\path.msi" Cache="keep" EmbedCab="yes"/>
  <MsiPackage SourceFile="$(BuildPath)\appendpath.msi" Cache="keep" EmbedCab="yes"/>
</Chain>
```

## 五、完整清理缓存 + 一键打包 64 位安装包

### 1. 清理旧编译产物（避免缓存冲突）

cmd







```
cd PCbuild
rmdir /s /q amd64 win32 obj
```

### 2. 执行官方打包脚本

cmd







```
cd ../Tools/msi
build.bat -x64
```

- 编译过程出现 `ICE69` 警告可直接忽略，**0 Error** 即打包成功；
- 打包耗时较长，取决于机器性能。

## 六、产物定位与交付使用

### 1. 最终单文件安装包路径

plaintext







```
PCbuild\amd64\en-us\python-3.11.15.9320-amd64.exe
```

### 2. 交付特性

1. 仅需复制这一个 exe 到任意全新 Windows 机器，无需附带任何 msi；
2. 内置 VC 运行库、Python 内核、标准库、tkinter 图形库、pip 包管理器；
3. 自带 py/pyw 启动器、右键.py 文件关联；
4. 安装后目录可自由移动盘符 / 路径，无裸编译文件夹 `is in build tree=1` 路径崩溃问题；
5. 安装界面支持自定义路径、自动添加系统 PATH。

### 3. 安装后验证（新开终端执行）

powershell







```
# 版本校验
python --version
# pip校验
python -m pip --version
# 核心扩展、tkinter校验
python -c "import encodings, _socket, tkinter; print('全部组件正常')"
# 交互exit校验
python
>>> exit()
```

## 七、历史踩坑根源总结（对比错误方案，理解本文流程必要性）

1. **错误路线 1：直接使用 PCbuild/amd64 绿色文件夹分发**

   根本缺陷：编译时硬写入源码绝对路径，迁移目录后底层源码树检测逻辑覆盖 `Py_BUILD_TREE=0` 宏，出现`encodings`、`_socket`模块丢失；添加`._pth`补丁会破坏 site 模块，`exit()`、pip 失效，补丁无法根治底层硬编码。

   最优替代：使用 MSI 安装包，自带 Windows 路径重定向逻辑，无迁移兼容问题。

2. **错误路线 2：不修改 launcher.wixproj 直接打包**

   报错根源：wixproj 自动并行编译三平台 pyshellext，独立 MSBuild 进程丢失 VS x64 工具集环境，报 MSB8020 找不到 v143；本文提前手动编译 64 位 dll 并删除自动编译 Target 彻底规避。

3. **错误路线 3：不裁剪 launcher_files.wxs 的 32/ARM64 节点**

   报错根源：WiX 链接阶段强制校验所有平台 dll 文件，仅编译 64 位会触发 LGHT0103 文件不存在；本文仅保留 amd64 组件消除校验依赖。

4. **错误路线 4：不开启 EmbedPayloads+EmbedCab**

   缺陷：输出 1MB 引导 exe，必须配套同目录全套 msi 才能安装，无法单文件交付客户 / 新机器。

## 八、可选体积精简优化（按需裁剪）

若不需要开发工具、测试套件、调试符号，可做两处精简减小 exe 体积：

1. 在 `snapshot.wxs` 注释不需要的 MsiPackage：

xml







```
<!-- <MsiPackage SourceFile="$(BuildPath)\dev.msi" Cache="keep" EmbedCab="yes"/> -->
<!-- <MsiPackage SourceFile="$(BuildPath)\test.msi" Cache="keep" EmbedCab="yes"/> -->
```

1. 打包完成后删除所有 `.wixpdb` 调试符号文件，不影响安装使用。