---
title: Android应用内安装App的几种姿势
date: 2017-10-27 17:16:58
tags: Android
categories: Android
---

#### 调用系统安装程序
这种方式最为简单，只需要调起系统界面即可。看代码

```java

    /**
     * 调用系统安装界面
     *
     * @param context
     * @param apkFile
     */
    public static void startInstallActivity(Context context, File apkFile) {
        if (apkFile == null || !apkFile.exists()) return;

        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        intent.setDataAndType(Uri.parse("file://" + apkFile.toString()), "application/vnd.android.package-archive");
        context.startActivity(intent);
    }
    
```

******

#### 调用系统pm命令
这种方式是在通过终端执行*pm*命令来实现 

```
pm install -r [filepath]
```
其中`-r`代表覆盖安装。 
这种方式可以实现静默安装，不需要调起系统界面。但是有一个前提，就是需要*root*权限，没有去*root*权限，*pm*命令不能执行。获取*root*权限不是本文范畴，请自行百度或者google。

满足前提的情况下，我们可能用到以下方法。

* 判断设备是否有root权限,通过执行su命令是否正确来检查设备是否被root

```java
    /**
     * 检查设备是否被root
     *
     * @return
     */
    public static boolean isRoot() {
        try {
            Process process = Runtime.getRuntime().exec("su");
            process.getOutputStream().write("exit\n".getBytes());
            process.getOutputStream().flush();
            int i = process.waitFor();
            if (0 == i) {
                process = Runtime.getRuntime().exec("su");
                return true;
            }

        } catch (Exception e) {
            return false;
        }
        return false;

    }
```
* 执行终端命令

```java
/**
     * 执行终端命令
     *
     * @param cmd
     * @return
     * @throws IOException
     * @throws InterruptedException
     */
    public static int execRootCmdSilent(String cmd) throws IOException, InterruptedException {
        int result = -1;
        DataOutputStream dos = null;

        try {
            Process p = Runtime.getRuntime().exec("su");
            dos = new DataOutputStream(p.getOutputStream());
            Log.i(TAG, cmd);
            dos.writeBytes(cmd + "\n");
            dos.flush();
            dos.writeBytes("exit\n");
            dos.flush();
            p.waitFor();
            result = p.exitValue();
        } finally {
            if (dos != null) {
                try {
                    dos.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return result;
    }
```
* 安装apk，在终端执行 `pm install -r filePath`

```java
 public static void installApk(String filePath) throws IOException, InterruptedException {
    if (TextUtils.isEmpty(filePath) || !new File(filePath).exists()) return;
    execRootCmdSilent("pm install -r " + filePath);
 }
```

******

#### 反射*PackageManager*的*installPackage*方法
通过查看查看系统安装程序的源码，可以看到系统安装程序实际上是调用*PackageManager*的*installPackage*方法来实现安装的。

```java
/**
     * @deprecated replaced by {@link PackageInstaller}
     * @hide
     */
    @Deprecated
    public abstract void installPackage(
            Uri packageURI, //APK地址
            IPackageInstallObserver observer, //安装回调
            @InstallFlags int flags,
            String installerPackageName); //被安装APK的包名
```

* *installPackage*方法虽然是*public*修饰的，但是同时被*@hide*了，所以应用程序无法直接调用，这就需要用到反射机制了。

* 调用这个方法需要系统权限，所以需要用系统签名对apk打包，具体操作可以参考[这篇文章](http://connorlin.github.io/2016/04/27/让Android-Studio支持系统签名&#40;证书&#41;)

* *IPackageInstallObserver*是一个*AIDL*接口，所以我们需要用到*IPackageInstallObserver.aidl* 这个文件。你可以

    * 直接从系统源码拷贝，该文件位于`/frameworks/base/core/java/android/content/pm`

    * 或者拷贝我的，在你的项目下创建`app/src/main/aidl/android/content/pm/IPackageInstallObserver.aidl`文件，然后将下面的代码复制进去

```java

package android.content.pm;

/**
 * API for installation callbacks from the Package Manager.
 * @hide
 */
oneway interface IPackageInstallObserver {
    void packageInstalled(in String packageName, int returnCode);
}


```

当上述都做好了之后。

* 调用安装的方法

```java
    //installPackage方法名
    private static final String INSTALL_METHOD = "installPackage";

    //InstallFlags
    public static final int INSTALL_FORWARD_LOCK = 0x00000001;
    public static final int INSTALL_REPLACE_EXISTING = 0x00000002;
    public static final int INSTALL_ALLOW_TEST = 0x0000000
    public static final int INSTALL_EXTERNAL = 0x00000008;
    public static final int INSTALL_INTERNAL = 0x00000010;
    public static final int INSTALL_FROM_ADB = 0x00000020;
    public static final int INSTALL_ALL_USERS = 0x00000040;
    public static final int INSTALL_ALLOW_DOWNGRADE = 0x00000080;
    public static final int INSTALL_GRANT_RUNTIME_PERMISSIONS = 0x00000100;
    public static final int INSTALL_FORCE_VOLUME_UUID = 0x00000200;
    public static final int INSTALL_FORCE_PERMISSION_PROMPT = 0x00000400;
    public static final int INSTALL_EPHEMERAL = 0x00000800;
    public static final int INSTALL_DONT_KILL_APP = 0x00001000;
    public static final int INSTALL_FORCE_SDK = 0x00002000;
    public static final int DONT_KILL_APP = 0x00000001;


    public static void installPackage(Context context, File file, IPackageInstallObserver observer)
            throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        PackageManager packageManager = context.getPackageManager();
        Method method = PackageManager.class.getDeclaredMethod(INSTALL_METHOD, Uri.class,
                IPackageInstallObserver.class, int.class, String.class);
        method.invoke(packageManager, Uri.fromFile(file), observer, INSTALL_REPLACE_EXISTING | INSTALL_DONT_KILL_APP, GetAppInfo.getAPKPackageName(context, file.getAbsolutePath()));
    }
    
```

* 监听安装回调

```java
    
    //安装回调
    private static class PackageInstallObserver extends IPackageInstallObserver.Stub {

        @Override
        public void packageInstalled(String packageName, int returnCode) throws RemoteException {
            if(returnCode == INSTALL_SUCCEEDED){
                //安装成功
            }
        }
    }
    
    
    //returnCode的取值
    public static final int INSTALL_SUCCEEDED = 1;
    public static final int INSTALL_FAILED_ALREADY_EXISTS = -1;
    public static final int INSTALL_FAILED_INVALID_APK = -2;
    public static final int INSTALL_FAILED_INVALID_URI = -3;
    public static final int INSTALL_FAILED_INSUFFICIENT_STORAGE = -4;
    public static final int INSTALL_FAILED_DUPLICATE_PACKAGE = -5;
    public static final int INSTALL_FAILED_NO_SHARED_USER = -6;
    public static final int INSTALL_FAILED_UPDATE_INCOMPATIBLE = -7;
    public static final int INSTALL_FAILED_SHARED_USER_INCOMPATIBLE = -8;
    public static final int INSTALL_FAILED_MISSING_SHARED_LIBRARY = -9;
    public static final int INSTALL_FAILED_REPLACE_COULDNT_DELETE = -10;
    public static final int INSTALL_FAILED_DEXOPT = -11;
    public static final int INSTALL_FAILED_OLDER_SDK = -12;
    public static final int INSTALL_FAILED_CONFLICTING_PROVIDER = -13;
    public static final int INSTALL_FAILED_NEWER_SDK = -14;
    public static final int INSTALL_FAILED_TEST_ONLY = -15;
    public static final int INSTALL_FAILED_CPU_ABI_INCOMPATIBLE = -16;
    public static final int INSTALL_FAILED_MISSING_FEATURE = -17;
    public static final int INSTALL_FAILED_CONTAINER_ERROR = -18;
    public static final int INSTALL_FAILED_INVALID_INSTALL_LOCATION = -19;
    public static final int INSTALL_FAILED_MEDIA_UNAVAILABLE = -20;
    public static final int INSTALL_FAILED_VERIFICATION_TIMEOUT = -21;
    public static final int INSTALL_FAILED_VERIFICATION_FAILURE = -22;
    public static final int INSTALL_FAILED_PACKAGE_CHANGED = -23;
    public static final int INSTALL_FAILED_UID_CHANGED = -24;
    public static final int INSTALL_FAILED_VERSION_DOWNGRADE = -25;
    public static final int INSTALL_FAILED_PERMISSION_MODEL_DOWNGRADE = -26;
    public static final int INSTALL_PARSE_FAILED_NOT_APK = -100;
    public static final int INSTALL_PARSE_FAILED_BAD_MANIFEST = -101;
    public static final int INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION = -102;
    public static final int INSTALL_PARSE_FAILED_NO_CERTIFICATES = -103;
    public static final int INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES = -104;
    public static final int INSTALL_PARSE_FAILED_CERTIFICATE_ENCODING = -105;
    public static final int INSTALL_PARSE_FAILED_BAD_PACKAGE_NAME = -106;
    public static final int INSTALL_PARSE_FAILED_BAD_SHARED_USER_ID = -107;
    public static final int INSTALL_PARSE_FAILED_MANIFEST_MALFORMED = -108;
    public static final int INSTALL_PARSE_FAILED_MANIFEST_EMPTY = -109;
    public static final int INSTALL_FAILED_INTERNAL_ERROR = -110;
    public static final int INSTALL_FAILED_USER_RESTRICTED = -111;
    public static final int INSTALL_FAILED_DUPLICATE_PERMISSION = -112;
    public static final int INSTALL_FAILED_NO_MATCHING_ABIS = -113;
    public static final int NO_NATIVE_LIBRARIES = -114;
    public static final int INSTALL_FAILED_ABORTED = -115;

```







