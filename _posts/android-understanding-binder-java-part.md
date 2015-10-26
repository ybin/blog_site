title: Android binder在Java层的使用方法
date: 2015-03-30 16:29:47
categories: android
tags: [android, java, binder]
---

通过一个实例来展示如何在Java层使用Android binder进行跨进程通讯。

<!-- more -->

### 概况

我们要实现一个计算器的例子，服务端用Service实现，它可以计算加减乘除等各种运算，
客户端是一个Activity，通过绑定到服务端，调用服务端的接口进行计算，然后把计算结果
展示出来(TextView)。

### 定义业务接口

对于我们这个例子来说，业务就是加减乘除各种计算，所以业务接口也就是计算器的各种接口。

```java ICalc.java
package com.ybin.calc.calculator;

import android.os.IBinder;
import android.os.IInterface;
import android.os.RemoteException;

public interface ICalc extends IInterface {
    // binder部分需要用到的常量
    static final String DESCRIPTOR = "com.example.calcservice";
    static final int TRANSACTION_add = (IBinder.FIRST_CALL_TRANSACTION + 0);

    // 业务接口
    int add(int a, int b) throws RemoteException;
}
```

### native部分

也就是service部分，真正的计算在这里进行，

```java CalcNative.java
package com.ybin.calc.calculator;

import android.os.Binder;
import android.os.IBinder;
import android.os.IInterface;
import android.os.Parcel;
import android.os.RemoteException;

public class CalcNative extends Binder implements ICalc {
    // 工具方法，应该放到单独的工具类里面，
    // 该方法只是做一个转换，与binder没有实质性的关联
    public static ICalc asInterface(IBinder obj) {
        IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if(iin != null) {
            return (ICalc)iin;
        }
        return new CalcProxy(obj);
    }

    // binder接口实现
    @Override
    public IBinder asBinder() {
        return this;
    }

    @Override
    protected boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_add: {
            data.enforceInterface(DESCRIPTOR);
            int a = data.readInt();
            int b = data.readInt();
            int result = this.add(a, b);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }
        }
        return super.onTransact(code, data, reply, flags);
    }

    // 业务接口实现
    @Override
    public int add(int a, int b) {
        return a + b;
    }
}
```

### proxy部分

也就是客户端的业务接口，它通过代理模式对binder的proxy部分进行打包，

```java CalcProxy.java
package com.ybin.calc.calculator;

import android.os.IBinder;
import android.os.Parcel;
import android.os.RemoteException;

public class CalcProxy implements ICalc {
    
    private IBinder mRemote;
    
    public CalcProxy(IBinder remote) {
        mRemote = remote;
    }

    @Override
    public IBinder asBinder() {
        return mRemote;
    }

    // 业务接口打包
    @Override
    public int add(int a, int b) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        int result = 0;
        
        try {
            data.writeInterfaceToken(DESCRIPTOR);
            data.writeInt(a);
            data.writeInt(b);
            mRemote.transact(TRANSACTION_add, data, reply, 0);
            reply.readException();
            result = reply.readInt();
        } finally {
            data.recycle();
            reply.recycle();
        }
        
        return result;
    }
}
```

### 实现Service

我们通过Service来提供服务，

```java
package com.ybin.calc.calcservice;

import com.ybin.calc.calculator.CalcNative;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;

public class CalcService extends Service {

    @Override
    public IBinder onBind(Intent intent) {
        return new CalcNative();
    }

}
```

### 实现client

客户端就是一个简单的activity而已，

```java
package com.ybin.calc.calcclient;

import android.app.Activity;
import android.content.ComponentName;
import android.content.Intent;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.os.RemoteException;
import android.util.Log;
import android.widget.TextView;

import com.ybin.calc.R;
import com.ybin.calc.calculator.CalcNative;
import com.ybin.calc.calculator.ICalc;

public class CalcClient extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.calcclient_main);


        ServiceConnection conn = new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                ICalc calc = CalcNative.asInterface(service);
                int r = 0;
                try {
                    r = calc.add(1, 2);
                } catch (RemoteException e) {
                    Log.e("CalcClient", "call service interface error");
                    e.printStackTrace();
                }
                TextView tv = (TextView) findViewById(R.id.text);
                tv.setText("1 + 2 = " + r);
            }

            @Override
            public void onServiceDisconnected(ComponentName name) {
            }
        };
        Intent intent = new Intent("com.ybin.calc.service");

        bindService(intent, conn, BIND_AUTO_CREATE);
    }
}
```

OK，现在运行即可。不过你可能会有疑问，client和service在同一个进程中啊，这哪里是进程间通讯啊，
好，我们设置一下manifest文件，

```xml
<service
    android:name=".calcservice.CalcService"
    android:process="com.ybin.calc.service">
    <intent-filter>
        <action android:name="com.ybin.calc.service" />
    </intent-filter>
</service>
```

现在可以进行IPC了。

其实，android已经提供了AIDL来简化这个过程，你只需定义aidl文件，即业务逻辑，proxy, native部分的
binder内容自动生成。

本文源码：[github][calc]

[calc]: https://github.com/ybin/AndroidDemo/tree/master/calc

(over)