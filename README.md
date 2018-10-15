# Android上基于NDK级别的安全认证方案

主要基于Cmake来构建本NDK级别的安全认证方案

# 主要解决的问题

* 防止别人用新的签名文件对APK源码进行二次打包
* 进一步保护应用的安全性，用NDK实现校验逻辑相比用Java实现更为安全，严谨
* 同时也可以保护SO库自身的安全性，防止JNI方法被其它应用调用

# 实现步骤

* 预设签名：在NDK层预先设定该应用的签名信息串
* 反射调用：在NDK层通过反射动态去获取当前应用的签名信息，与预设签名进行比较
* 如又要兼备保护so库的jni方法不被其它App调用，又要有签名校验的功能，则需要在C++代码上加上JNI_OnLoad 逻辑：

```java
/**
 * 当so库被初始化的时候，手动去调用JNI方法时，该方法被调用
 */
jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = NULL;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
        return JNI_ERR;
    }
    if (verifySign(env) == JNI_OK) {
        return JNI_VERSION_1_4;
    }
//    LOGE("签名不一致!");
    return JNI_ERR;
}

```

* 【Java端调用】直接调用SecurityCore->isSecuritySign()的native方法进行签名合法性校验

```java
SecurityCore.isSecuritySign(); //返回true:签名合法；false：不合法
```

# 注意

* 使用前需要把 NDKSecurityLibrary/src/main/cpp/SecurityCore.cpp 文件里的预设签名*SIGN换成你自己应用的签名信息
* 预设签名可以用Java代码事先获取

```java
    /**
     * 展示了如何用Java代码获取签名
     */
    private String getSign() {
        try {
            // 下面几行代码展示如何任意获取Context对象，在jni中也可以使用这种方式
//            Class<?> activityThreadClz = Class.forName("android.app.ActivityThread");
//            Method currentApplication = activityThreadClz.getMethod("currentApplication");
//            Application application = (Application) currentApplication.invoke(null);
//            PackageManager pm = application.getPackageManager();
//            PackageInfo pi = pm.getPackageInfo(application.getPackageName(), PackageManager.GET_SIGNATURES);

            PackageManager pm = getPackageManager();
            PackageInfo pi = pm.getPackageInfo(getPackageName(), PackageManager.GET_SIGNATURES);
            Signature[] signatures = pi.signatures;
            Signature signature0 = signatures[0];
            return signature0.toCharsString();
        } catch (Exception e) {
            e.printStackTrace();
            return "";
        }
    }
```
















