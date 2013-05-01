---
layout: post
title: "Parcel object in android"
date: 2012-11-03 21:42
comments: true
categories: Android
---

还记得之前被Android Framework里面的那个Parcel类搞得很摸不清头脑，这次为了完成一个任务把它彻底理了一遍，终于弄清楚了很多。

要做的事是这样子的：我有一个经过Parcel封装过的数据data，我需要通过socket把它发送到远程的一个进程中去进行处理，再返回一个Parcel对象reply。

首先在做这个之前我们先来看下Parcel在android中的用法和实现：

#####用法

{% codeblock lang:java %}
public java.lang.String sayHello(java.lang.String message) throws android.os.RemoteException
{
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    java.lang.String _result;
    try {
	_data.writeInterfaceToken(DESCRIPTOR);
	_data.writeString(message);
	mRemote.transact(Stub.TRANSACTION_sayHello, _data, _reply, 0);
	_reply.readException();
	_result = _reply.readString();
    }
}
{% endcodeblock %}

<!-- more -->

这是一段用的最多的code，我们在android中的BinderProxy对象中调用一个sayHello()方法，它会首先封装一个Parcel对象，并将其作为参数调用transact()函数，通过android的Binder机制传到另一个进程的BBinder的onTransact()函数中：

{% codeblock lang:java %}
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
    switch (code)
    {
	case INTERFACE_TRANSACTION:
	    {
		reply.writeString(DESCRIPTOR);
		return true;
	    }
	case TRANSACTION_sayHello:
	    {
		data.enforceInterface(DESCRIPTOR);
		java.lang.String _arg0;
		_arg0 = data.readString();
		java.lang.String _result = this.sayHello(_arg0);
		reply.writeNoException();
		reply.writeString(_result);
		return true;
	    }
    }
    return super.onTransact(code, data, reply, flags);
}
{% endcodeblock %}

而这两个进程数据的传递是在driver里面通过内存拷贝进行的。

那么如果我希望不通过binder driver而直接将Parcel对象data通过socket传到另一个进程那么该怎么办呢？大家都知道在Java里面要传递一个数据，该数据必须得是Serializable的。那么Parcel类是这样的吗？

需要先来看看Parcel的实现:

#####实现

{% codeblock lang:java %}
public final class Parcel {
    ...
    public final native int dataSize();
    ...
    public final native int dataPosition();
    ...
    public final native byte[] marshall();
    ...
    public final native void unmarshall(byte[] data, int offest, int length);
    ...
    public final native void writeInterfaceToken(String interfaceName);
    ...
    public final native void enforceInterface(String interfaceName);
    ...
    public final native void writeString(String val);
    ...
    public final native String readString();
    ...
}
{% endcodeblock %}

首先可以看到Parcel并没有implements Serializable，同时它的方法都是JNI方法，可以在frameworks/base/core/jni/android_util_Binder.cpp里面找到，这里以writeString()为例：

{% codeblock lang:c %}
static void android_os_Parcel_writeString(JNIEnv* env, jobject clazz, jstring val)
{
    Parcel* parcel = parcelForJavaObject(env, clazz);
    if (parcel != NULL) {
        status_t err = NO_MEMORY;
        if (val) {
            const jchar* str = env->GetStringCritical(val, 0);
            if (str) {
                err = parcel->writeString16(str, env->GetStringLength(val));
                env->ReleaseStringCritical(val, str);
            }
        } else {
            err = parcel->writeString16(NULL, 0);
        }
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
{% endcodeblock %}

里面调用了Parcel的writeString16()这个方法，而这个Parcel是一个C类，实现在frameworks/base/libs/binder/Parcel.cpp里面：

{% codeblock lang:c %}
...
status_t Parcel::writeString16(const char16_t* str, size_t len)
{
    if (str == NULL) return writeInt32(-1);
    
    status_t err = writeInt32(len);
    if (err == NO_ERROR) {
        len *= sizeof(char16_t);
        uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
        if (data) {
            memcpy(data, str, len);
            *reinterpret_cast<char16_t*>(data+len) = 0;
            return NO_ERROR;
        }
        err = mError;
    }
    return err;
}

...

status_t Parcel::writeInt32(int32_t val)
{
    return writeAligned(val);
}

...

template<class T>
status_t Parcel::writeAligned(T val) {
    COMPILE_TIME_ASSERT_FUNCTION_SCOPE(PAD_SIZE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }

    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}

...

void* Parcel::writeInplace(size_t len)
{
    const size_t padded = PAD_SIZE(len);

    // sanity check for integer overflow
    if (mDataPos+padded < mDataPos) {
        return NULL;
    }

    if ((mDataPos+padded) <= mDataCapacity) {
restart_write:
        //printf("Writing %ld bytes, padded to %ld\n", len, padded);
        uint8_t* const data = mData+mDataPos;

        // Need to pad at end?
        if (padded != len) {
#if BYTE_ORDER == BIG_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0xffffff00, 0xffff0000, 0xff000000
            };
#endif
#if BYTE_ORDER == LITTLE_ENDIAN
            static const uint32_t mask[4] = {
                0x00000000, 0x00ffffff, 0x0000ffff, 0x000000ff
            };
#endif
            //printf("Applying pad mask: %p to %p\n", (void*)mask[padded-len],
            //    *reinterpret_cast<void**>(data+padded-4));
            *reinterpret_cast<uint32_t*>(data+padded-4) &= mask[padded-len];
        }

        finishWrite(padded);
        return data;
    }

    status_t err = growData(padded);
    if (err == NO_ERROR) goto restart_write;
    return NULL;
}
{% endcodeblock %}

其实整个Parcel的实现是这样的，每一个Parcel对象有一个指针mData，指向一块内存的起始地址，一个指针mDataPos，指向到目前未知已经读到的数据的大小。在写一个String的时候，首先将字符串的大小通过writeInt32()写到某个合适的内存地址中：

{% codeblock lang:c %}
    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
        return finishWrite(sizeof(val));
    }
{% endcodeblock %}

然后通过writeInplace将字符串按照合适的Align方式写到之后的内存地址中：

{% codeblock lang:c %}
    uint8_t* data = (uint8_t*)writeInplace(len+sizeof(char16_t));
    if (data) {
	memcpy(data, str, len);
	*reinterpret_cast<char16_t*>(data+len) = 0;
	return NO_ERROR;
    }
{% endcodeblock %}

最后更新相对应的mDataSize和mDataPos值。

------

######Parcel对象的序列化

也就是说我们在java层是没有办法不通过其提供的接口获得一个Parcel对象内存中的数据的，也就是说如果我们需要将Parcel对象通过socket传输，就只有两种方式：

* 在jni层进行socket传输；
* 手动对其进行序列化，并将序列化的数据通过socket传输过去。

其实细心的人可以发现我刚刚在列举Parcel的一系列jni方法的时候还列举了两个方法：

{% codeblock lang:java %}
public final class Parcel {
    ...
    public final native byte[] marshall();
    ...
    public final native void unmarshall(byte[] data, int offest, int length);
    ...
}
{% endcodeblock %}

这两个方法在jni里面是这样实现的：

{% codeblock lang:c %}
static jbyteArray android_os_Parcel_marshall(JNIEnv* env, jobject clazz)
{
    Parcel* parcel = parcelForJavaObject(env, clazz);
    if (parcel == NULL) {
       return NULL;
    }

    // do not marshall if there are binder objects in the parcel
    if (parcel->objectsCount())
    {
        jniThrowException(env, "java/lang/RuntimeException", "Tried to marshall a Parcel that contained Binder objects.");
        return NULL;
    }

    jbyteArray ret = env->NewByteArray(parcel->dataSize());

    if (ret != NULL)
    {
        jbyte* array = (jbyte*)env->GetPrimitiveArrayCritical(ret, 0);
        if (array != NULL)
        {
            memcpy(array, parcel->data(), parcel->dataSize());
            env->ReleasePrimitiveArrayCritical(ret, array, 0);
        }
    }

    return ret;
}

static void android_os_Parcel_unmarshall(JNIEnv* env, jobject clazz, jbyteArray data, jint offset, jint length)
{
    Parcel* parcel = parcelForJavaObject(env, clazz);
    if (parcel == NULL || length < 0) {
       return;
    }

    jbyte* array = (jbyte*)env->GetPrimitiveArrayCritical(data, 0);
    if (array)
    {
        parcel->setDataSize(length);
        parcel->setDataPosition(0);

        void* raw = parcel->writeInplace(length);
        memcpy(raw, (array + offset), length);

        env->ReleasePrimitiveArrayCritical(data, array, 0);
    }
}
{% endcodeblock %}

marshall和unmarshall正是两个Parcel提供的将其自身进行序列化和反序列化的方法，marshall()将其本身转换成一个byte[]数据序列，即将mData指向内存中的值通过内存拷贝的形式传到一个byte[]变量中去并返回，而unmarshall()则是将byte[]变量中的数据拷贝到Parcel对象的mData指向内存中。

也就是说，我们可以通过marshall方法将data序列化成byte[]变量dataBytes，通过socket的write传递到远端，在远端通过unmarshall方法将其转换回来。

另外通过实现，发现还有一个需要注意的地方，在远端通过unmarshall对象转换回来的时候，还需要调用data.setDataPosition(0)将mDataPos设成初始未知，否则之后无法读取正常的数据。
