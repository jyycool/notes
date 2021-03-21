## Mac安装MAT1.9.2报错解决

### 错误日志

下载Eclipse Memory Analyzer在mac上打开的时候出现以下异常：

打开提示信息中的日志文件,错误内容如下:

```ceylon
!SESSION 2018-05-03 12:36:38.607 -----------------------------------------------
eclipse.buildId=unknown
java.version=1.8.0_161
java.vendor=Oracle Corporation
BootLoader constants: OS=macosx, ARCH=x86_64, WS=cocoa, NL=zh_CN
Framework arguments:  -keyring /Users/sherlock/.eclipse_keyring
Command-line arguments:  -os macosx -ws cocoa -arch x86_64 -keyring /Users/sherlock/.eclipse_keyring

!ENTRY org.eclipse.osgi 4 0 2018-05-03 12:36:45.892
!MESSAGE Application error
!STACK 1
java.lang.IllegalStateException: The platform metadata area could not be written: /private/var/folders/6p/dn3tmpz1043_3gfrg8v3hdtr0000gn/T/AppTranslocation/D18C449A-CBE3-41CB-9BD4-20DC865B3C2F/d/mat.app/Contents/MacOS/workspace/.metadata.  By default the platform writes its content
under the current working directory when the platform is launched.  Use the -data parameter to
specify a different content area for the platform.
        at org.eclipse.core.internal.runtime.DataArea.assertLocationInitialized(DataArea.java:70)
        at org.eclipse.core.internal.runtime.DataArea.getStateLocation(DataArea.java:138)
        at org.eclipse.core.internal.preferences.InstancePreferences.getBaseLocation(InstancePreferences.java:44)
        at org.eclipse.core.internal.preferences.InstancePreferences.initializeChildren(InstancePreferences.java:209)
        at org.eclipse.core.internal.preferences.InstancePreferences.<init>(InstancePreferences.java:59)
        at org.eclipse.core.internal.preferences.InstancePreferences.internalCreate(InstancePreferences.java:220)
        at org.eclipse.core.internal.preferences.EclipsePreferences.create(EclipsePreferences.java:349)
        at org.eclipse.core.internal.preferences.EclipsePreferences.create(EclipsePreferences.java:337)
        at org.eclipse.core.internal.preferences.PreferencesService.createNode(PreferencesService.java:393)
        at org.eclipse.core.internal.preferences.RootPreferences.getChild(RootPreferences.java:60)
        at org.eclipse.core.internal.preferences.RootPreferences.getNode(RootPreferences.java:95)
        at org.eclipse.core.internal.preferences.RootPreferences.node(RootPreferences.java:84)
        at org.eclipse.core.internal.preferences.AbstractScope.getNode(AbstractScope.java:38)
        at org.eclipse.core.runtime.preferences.InstanceScope.getNode(InstanceScope.java:77)
        at org.eclipse.ui.preferences.ScopedPreferenceStore.getStorePreferences(ScopedPreferenceStore.java:225)
        at org.eclipse.ui.preferences.ScopedPreferenceStore.<init>(ScopedPreferenceStore.java:132)
        at org.eclipse.ui.plugin.AbstractUIPlugin.getPreferenceStore(AbstractUIPlugin.java:287)
        at org.eclipse.ui.internal.Workbench.lambda$3(Workbench.java:609)
        at org.eclipse.core.databinding.observable.Realm.runWithDefault(Realm.java:336)
        at org.eclipse.ui.internal.Workbench.createAndRunWorkbench(Workbench.java:597)
        at org.eclipse.ui.PlatformUI.createAndRunWorkbench(PlatformUI.java:148)
        at org.eclipse.mat.ui.rcp.Application.start(Application.java:26)
        at org.eclipse.equinox.internal.app.EclipseAppHandle.run(EclipseAppHandle.java:196)
        at org.eclipse.core.runtime.internal.adaptor.EclipseAppLauncher.runApplication(EclipseAppLauncher.java:134)
"1588480598922.log" 48L, 3830C
```

### 原因

`/private/var/folders/6p/dn3tmpz1043_3gfrg8v3hdtr0000gn/T/AppTranslocation/D18C449A-CBE3-41CB-9BD4-20DC865B3C2F/d/mat.app/Contents/MacOS/workspace/.metadata`是只读文件

### 解决

1. 邮件点击mat.app，选择显示包内容

2. 进入Contents/Eclipse文件夹，打开 MemoryAnalyzer.ini

![](/Users/sherlock/Desktop/notes/allPics/Java/mat.png)

3. 添加data相关信息(即数据保存路径)

```
-data
/Users/sherlock/environment/mat/log
```

4. 再次打开mat即可正常使用

- 注意事项
  - data参数和路径必须在两个不同的行
  - data参数必须放在Laucher之前，否则启动还是不成功