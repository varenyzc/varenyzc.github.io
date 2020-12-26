---
title: 黑科技——静默卸载、清除数据及降级安装
date:  2020-12-22 18:39:45 +0800
category: develop 
tags: Android
excerpt: 工作中的一些小总结。
---

> 近期在做微信数据备份项目时，所探索出来的一些“黑科技”，可以实现静默卸载第三方app、静默清除第三方app的数据、静默降级安装app等功能

**首先这些都需要系统签名才有这个能力，其次，通过反射去调用系统的隐藏api去实现这些功能。**隐藏api，顾名思义，普通情况下肯定是调用不到的，所以需要一些特殊的方法去调用它。

### 0.反射方法
这里先提供反射的通用方法，后面示例代码将从这里调用，可做一个util工具类使用。
```java
public class ReflectUtil {
    // 通过反射去调用target类下的实例方法
    public static Object callObjectMethod(Object target, String method, Class<?>[] parameterTypes, Object... values)
            throws NoSuchMethodException, SecurityException, IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        Class<? extends Object> clazz = target.getClass();
        Method declaredMethod = clazz.getDeclaredMethod(method, parameterTypes);
        return declaredMethod.invoke(target, values);
    }

  //通过反射去拿到实例对象
    public static Object callStaticObjectMethod(Class<?> clazz, String method, Class<?>[] parameterTypes, Object... values)
            throws NoSuchMethodException, SecurityException, IllegalAccessException, IllegalArgumentException, InvocationTargetException {
        Method declaredMethod = clazz.getDeclaredMethod(method, parameterTypes);
        declaredMethod.setAccessible(true);
        return declaredMethod.invoke(null, values);
    }
}
```
---

### 1.卸载
首先安装卸载的相关逻辑存在系统的framework层，路径为/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java，即PMS。
通过翻android源码（这里推荐一个在线看源码的网站http://androidxref.com/），可以知道卸载的方法为deletePackageAsUser，如下图：
![deletePackageAsUser部分源码](https://upload-images.jianshu.io/upload_images/13517457-a21bff75877bc171.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到该方法所需参数有packageName、versionCode、observer（通过aidl获取该类，后续会提到）、userId、flags。

参数中的IPackageDeleteObserver类，是一个aidl回调通知接口，在源码中找到这个接口：
```java
package android.content.pm;

/**
 * API for deletion callbacks from the Package Manager.
 *
 * {@hide}
 */
oneway interface IPackageDeleteObserver {
    void packageDeleted(in String packageName, in int returnCode);
}
```

将该aidl文件按照以上Package：android.content.pm的层级关系放置于项目源码中，会自动生成class文件供调用。生成文件如下图：
![aidl生成的类文件](https://upload-images.jianshu.io/upload_images/13517457-60c0845dccd2f315.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
成功生成这个之后，就可以使用反射实现卸载操作了。直接上代码：
```java
//卸载操作
  try {
          // 拿到PackageName对应的PackageInfo
            PackageInfo packageInfo = context.getPackageManager().getPackageInfo(yourPackageName, 0);
            PackageDeleteObserver observer = new PackageDeleteObserver();
           //通过反射拿到IBinder对象
            IBinder pgmService = (IBinder) ReflectUtil.callStaticObjectMethod(Class.forName("android.os.ServiceManager"), "getService", new Class[]{String.class}, "package");
           //通过反射拿到IPackageManager对象 
           Object mIPm = ReflectUtil.callStaticObjectMethod(Class.forName("android.content.pm.IPackageManager$Stub"),
                    "asInterface", new Class[]{IBinder.class}, pgmService);
           //通过反射使用IPackageManager接口调用到PackageManagerService里的deletePackageAsUser方法
          //其中后面两个0，第一个是传入userId，一般传入0即可，不同的系统会有不同的分身userid，第二个是flag，也是传入0即可。
            ReflectUtil.callObjectMethod(mIPm, "deletePackageAsUser",
                    new Class[]{String.class, int.class, IPackageDeleteObserver.class, int.class, int.class},
                    yourPackageName, packageInfo.versionCode, observer, 0, 0);
        } catch (Exception e) {
            
        }


//重写IPackageDeleteObserver的packageDeleted方法，监听应用卸载情况
private class PackageDeleteObserver extends IPackageDeleteObserver.Stub {

        @Override
        public void packageDeleted(String packageName, int returnCode) {
            Log.d(TAG, "packageName:"+packageName+" returnCode:"+returnCode);
            if (returnCode == 1) {
                //returnCode = 1即成功
            }
        }
    }
```

然后再在AndroidManifest.xml文件里添加两行权限声明：
```java
    <uses-permission android:name="android.permission.DELETE_PACKAGES"/>
    <uses-permission android:name="android.permission.INTERACT_ACROSS_USERS_FULL" />
```
即可静默卸载第三方app。

---

### 2.清除数据
清除数据同样是调用了PackageManagerService里的方法，方法名为clearApplicationUserData，源码如下：
![clearApplicationUserData部分源码](https://upload-images.jianshu.io/upload_images/13517457-6ca39e13d4693b6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到该方法需要传入packageName、IPackageDataObserver、userId等参数，其中IPackageDataObserver同上面的IPackageDeleteObserver接入方法，代码如下：
```java
package android.content.pm;

/**
 * API for package data change related callbacks from the Package Manager.
 * Some usage scenarios include deletion of cache directory, generate
 * statistics related to code, data, cache usage(TODO)
 * {@hide}
 */
oneway interface IPackageDataObserver {
    void onRemoveCompleted(in String packageName, boolean succeeded);
}
```
编译后可正常拿到IPackageDataObserver.class，接下来即可通过反射调用clearApplicationUserData方法了，代码如下：
```java
//清数据操作
  try {
            DeleteUserDataObserver observer = new DeleteUserDataObserver();
            IBinder pgmService = (IBinder) ReflectUtil.callStaticObjectMethod(Class.forName("android.os.ServiceManager"),
                    "getService", new Class[]{String.class}, "package");
            Object mIPm = ReflectUtil.callStaticObjectMethod(Class.forName("android.content.pm.IPackageManager$Stub"),
                    "asInterface", new Class[]{IBinder.class}, pgmService);
            ReflectUtil.callObjectMethod(mIPm, "clearApplicationUserData",
                    new Class[]{String.class, IPackageDataObserver.class, int.class},
                    yourPackageName, observer, 0);
        } catch (Exception e) {
         
        }

//实现IPackageDataObserver
private class DeleteUserDataObserver extends IPackageDataObserver.Stub {
        @Override
        public void onRemoveCompleted(String packageName, boolean succeeded) {
            Log.d(TAG, "packageName:" + packageName + " isSuccess:" + succeeded);
        }
    }
```

此操作需要声明如下权限：
```java
<uses-permission android:name="android.permission.CLEAR_APP_USER_DATA" />
```

---

### 3.降级安装
这个就厉害了，在leader的启示下，原来adb命令都是在源码工程中能找到对应的执行方法。adb中有一个命令可以实现降级安装adb install -r -d
![adb install](https://upload-images.jianshu.io/upload_images/13517457-952e041c2e1375f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过在源码中溯源，具体过程就不展示了，最终调用了PackageInstaller.SessionParams中的setRequestDowngrade方法，具体如下图：
![setRequestDowngrade](https://upload-images.jianshu.io/upload_images/13517457-953642170264ed1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到这同样是一个hide的api，同样需要用反射去调用。在安装过程中拿到SessionParams的实例对象，然后使用反射调用上述方法，传入true，即可实现降级安装，具体安装流程可参考其他博客，这里给出反射代码：
```java
PackageInstaller.SessionParams sp =
                    new PackageInstaller.SessionParams(PackageInstaller.SessionParams.MODE_FULL_INSTALL);
            ReflectUtil.callObjectMethod(sp, "setRequestDowngrade",
                    new Class[]{boolean.class}, true);
```

---

### 4.总结
**源码是最好的老师**，当有卡壳的地方时，看源码往往能解决99%的问题。例如在静默卸载过程中，我们app已经成功调用了反射方法，且没有catch到异常，但是一直没实现效果，且observer中没有打印出东西，后面看了miui的源码，发现在miui的framework层设置了一个白名单，只有白名单内的app有权限去调用卸载方法，于是在白名单中增加了我们的app，编译后即可正常实现效果。





