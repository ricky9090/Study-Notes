在学习Android Framework源码时，发现部分类中声明了这个变量：
```java
private int mPtr; // used by native code
```

那么native层是怎么使用这个变量的呢？
以Framework中的``MessageQueue``为例，它的Native模块源码为``/frameworks/base/core/jni/android_os_MessageQueue.cpp``
查看其中的代码，可以发现这几个调用：
```cpp
// Java 层的MessageQueue构造函数中，会调用Native中的这个初始化函数
static void android_os_MessageQueue_nativeInit(JNIEnv* env, jobject obj) {
    // 在Native中，创建了一个NativeMessageQueue对象
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (! nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return;
    }

    // 调用set方法，把创建的Natvie对象传递进去
    android_os_MessageQueue_setNativeMessageQueue(env, obj, nativeMessageQueue);
}

// set方法
static void android_os_MessageQueue_setNativeMessageQueue(JNIEnv* env, jobject messageQueueObj,
        NativeMessageQueue* nativeMessageQueue) {
    // 把native中的对象地址，赋值给Java层mPtr
    env->SetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr,
             reinterpret_cast<jint>(nativeMessageQueue));
}

// get方法
static NativeMessageQueue* android_os_MessageQueue_getNativeMessageQueue(JNIEnv* env,
        jobject messageQueueObj) {
    // 取到Java中的mPtr值
    jint intPtr = env->GetIntField(messageQueueObj, gMessageQueueClassInfo.mPtr);
    // 由于之前是把native对象地址放进去，这次再次通过reinterpret_cast把地址转换回native对象引用
    return reinterpret_cast<NativeMessageQueue*>(intPtr);
}
```

参考上面注释，可以发现``mPtr``是用来保存Native层生成对象的引用。那么在后续调用Native中的方法时，就可以把地址转换回Native对象了。

在``/frameworks/base/core/jni/android_os_MessageQueue.cpp``中，可以看到get方法的调用：
```cpp
sp<Looper> android_os_MessageQueue_getLooper(JNIEnv* env, jobject messageQueueObj) {
    // 使用get方法，从Java对象中拿到mPtr地址，转换成Native对象引用
    NativeMessageQueue* nativeMessageQueue =
            android_os_MessageQueue_getNativeMessageQueue(env, messageQueueObj);
    return nativeMessageQueue != NULL ? nativeMessageQueue->getLooper() : NULL;
}
```

我们可以模仿写一个Demo。

首先创建一个Java类：
```java
public class JniStudy {
    
    private long mPtr;  // Native层使用
    
    private native void nativeBindInfo();  // 绑定Java类信息
    
    public native void nativeCreateObj();  // 创建一个Natvie对象
    public native void nativePrintCacheMessage();  // 获取先前创建的Native对象，打印消息
    public native void nativeDestroy();  // 释放Natvie对象
    
    static {
        System.load("/home/myname/spc-work/jni-test-2/libjnistudy");
    }
    
    public JniStudy() {
        nativeBindInfo();
        nativeCreateObj();
    }
    
    public void printPtr() {
        System.out.println("Java: mPtr = " + this.mPtr);
    }
    
    public void onDestroy() {
        this.nativeDestroy();
    }

    public static void main(String[] args) {
        JniStudy test = new JniStudy();
        
        test.printPtr();
        test.nativePrintCacheMessage();
        
        test.onDestroy();
        
        test.printPtr();
    }
}
```

在这里我们定义了四个Native方法，具体作用见注释。接下来通过命令生成头文件，然后编写Native层代码。由于涉及对象的创建，因此使用C++来编写：
```cpp
#include <iostream>
#include "JniStudy.h"
using namespace std;

// 创建一个结构体，保存Java类信息，便于后续使用
static struct {
    jclass clazz;
    jfieldID mPtr;
} gJniStudyClassInfo;

// Native中的对象类，包含一个打印方法
class NativeJniStudy {
public:
    NativeJniStudy();
    ~NativeJniStudy();
    
    void printNativeMsg();
};

NativeJniStudy::NativeJniStudy() {
    cout << "JNI:  NativeJniStudy create" << endl;
}

NativeJniStudy::~NativeJniStudy() {
    cout << "JNI:  NativeJniStudy destory" << endl;
}

// Native对象打印信息方法
void NativeJniStudy::printNativeMsg() {
	cout << "Hello message from a NativeJniStudy object !!!" << endl;
}

// get方法，从mPtr中得到对象引用
static NativeJniStudy* getNativeRef(JNIEnv* env, jobject obj) {
    jlong ptr = env->GetLongField(obj, gJniStudyClassInfo.mPtr);
    return reinterpret_cast<NativeJniStudy*>(ptr);
}

// set方法，把对象地址保存到mPtr中
static void setNativeRef(JNIEnv* env, jobject obj, NativeJniStudy* nativeObj) {
    env->SetLongField(
		obj, 
		gJniStudyClassInfo.mPtr, 
		reinterpret_cast<jlong>(nativeObj)
	);
}

// 把类信息保存到结构体变量中，方便后续使用
void Java_JniStudy_nativeBindInfo(JNIEnv *env, jobject obj) {
	cout << "JNI:  Bind Class Info" << endl;
	jclass _clz = env->FindClass("JniStudy");
	gJniStudyClassInfo.clazz = _clz;
	jfieldID fieldId = env->GetFieldID(_clz, "mPtr", "J");
	gJniStudyClassInfo.mPtr = fieldId;
}

// 创建Native对象并保存
void Java_JniStudy_nativeCreateObj(JNIEnv *env, jobject obj) {
	NativeJniStudy* nativeObj = new NativeJniStudy();
	setNativeRef(env, obj, nativeObj);
}

// 拿到之前保存的对象，调用打印信息方法
void Java_JniStudy_nativePrintCacheMessage(JNIEnv *env, jobject obj) {
	cout << "JNI:  Java_JniStudy_nativePrintCacheMessage :" << endl;
	NativeJniStudy* _obj = getNativeRef(env, obj);
	if (_obj != NULL) {
		_obj->printNativeMsg();
	} else {
		cout << "JNI:  getNativeRef NULL !!!" << endl;
	}
}

// 释放Native对象
void Java_JniStudy_nativeDestroy(JNIEnv *env, jobject obj) {
	cout << "JNI:  Java_JniStudy_nativeDestroy :" << endl;
	NativeJniStudy* _nativeObj = getNativeRef(env, obj);
	if (_nativeObj != NULL) {
		setNativeRef(env, obj, NULL);
		delete _nativeObj;
	} else {
		cout << "JNI:  getNativeRef NULL !!!" << endl;
	}
}
```

具体流程可见注释。

可以观察到，在上边的Demo源码中，``mPtr``为``long``型；而在最开始的Framework代码中是``int``型。实际上面的Framework代码是Android 4.x版本的，目前新版Android代码，``mPtr``也都改为``long``型了。

在旧版本时代，手机设备还都使用的32位系统，因此``int``型变量可以满足需求；但当前手机系统均为64位，如果Java层仍然使用``int``保存，那么数据范围就不够了。如果把Demo中的``mPtr``改成``int``型，那么在Linux 64位测试机上，GCC编译时就会抛出错误信息，提示``reinterpret_cast``转换到``int``后会丢失信息:``loses precision``。

将Demo代码编译成功后运行，可看到运行结果：
```
JNI:  Bind Class Info
JNI:  NativeJniStudy create
Java: mPtr = 139764679171648
JNI:  Java_JniStudy_nativePrintCacheMessage :
Hello message from a NativeJniStudy object !!!
JNI:  Java_JniStudy_nativeDestroy :
JNI:  NativeJniStudy destory
Java: mPtr = 0
```

可以看到Native对象创建后，成功的保存在mPtr中；
然后通过JNI调用，成功的运行了之前保存的Native对象方法；
在释放Native对象后，将mPtr内容清空，它的值也变为0。