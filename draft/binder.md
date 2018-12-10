先看lib的实现：
```java
// IFibonacciService.aidl
package com.zgq.android.fibonaccicommon;
import com.zgq.android.fibonaccicommon.FibonacciRequest;
import com.zgq.android.fibonaccicommon.FibonacciResponse;

interface IFibonacciService {
  long fibJR(in long n);
  long fibJI(in long n);
  FibonacciResponse fib(in FibonacciRequest request);
}
``
下面两个类是aidl文件所依赖的。
```java
//FibonacciRequest.java
package com.zgq.android.fibonaccicommon;

import android.os.Parcel;
import android.os.Parcelable;

/**
 * Created by zgq on 2018/3/16.
 */

public class FibonacciRequest implements Parcelable{
    public  static enum Type{
        RECURSIVE_JAVA, ITERATIVE_JAVA
    }
    private final long n;
    private final Type type;

    public FibonacciRequest(long n, Type type){
        this.n = n;
        if(type == null){
            throw new NullPointerException("Type must not be null");
        }
        this.type = type;
    }

    public long getN() {
        return n;
    }

    public Type getType() {
        return type;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeLong(this.n);
        parcel.writeInt(this.type.ordinal());
    }
    public static final Creator<FibonacciRequest> CREATOR = new Creator<FibonacciRequest>(){
        public FibonacciRequest createFromParcel(Parcel in){
            long n = in.readLong();
            Type type = Type.values()[in.readInt()];
            return new FibonacciRequest(n,type);
        }
        public FibonacciRequest[] newArray(int size){
            return  new FibonacciRequest[size];
        }

};
}
```
```java
//FibonacciResponse.java
package com.zgq.android.fibonaccicommon;

import android.os.Parcel;
import android.os.Parcelable;

/**
 * Created by zgq on 2018/3/16.
 */

public class FibonacciResponse implements Parcelable{

    private final long result;
    private final long timeInMullis;

    public FibonacciResponse(long result, long timeInMullis) {
        this.result = result;
        this.timeInMullis = timeInMullis;
    }

    public long getResult() {
        return result;
    }

    public long getTimeInMullis() {
        return timeInMullis;
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeLong(this.result);
        parcel.writeLong(this.timeInMullis);
    }
    public static final Creator<FibonacciResponse> CREATOR = new Creator<FibonacciResponse>(){
        public FibonacciResponse createFromParcel(Parcel in){
            return  new FibonacciResponse(in.readLong(),in.readLong());
        }
        public FibonacciResponse[] newArray (int size){
            return  new FibonacciResponse[size];
        }
    };
}
```
FibonacciResponse和FibonacciRequest同样需要定义为aidl。
```java
//FibonacciRequest.aidl
package com.zgq.android.fibonaccicommon;
parcelable FibonacciRequest;
//FibonacciResponse.aidl
package com.zgq.android.fibonaccicommon;
parcelable FibonacciResponse;
```
编译一下就可以生成IFabnacciService.java类：
```java
package com.zgq.android.fibonaccicommon;
public interface IFibonacciService extends android.os.IInterface {
    public static abstract class Stub extends android.os.Binder implements com.zgq.android.fibonaccicommon.IFibonacciService {
        private static final java.lang.String DESCRIPTOR = "com.zgq.android.fibonaccicommon.IFibonacciService";

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        public static com.zgq.android.fibonaccicommon.IFibonacciService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.zgq.android.fibonaccicommon.IFibonacciService))) {
                return ((com.zgq.android.fibonaccicommon.IFibonacciService) iin);
            }
            return new com.zgq.android.fibonaccicommon.IFibonacciService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
          ...
            }
        }
    private static class Proxy implements com.zgq.android.fibonaccicommon.IFibonacciService {
            private android.os.IBinder mRemote;
android.os.RemoteException {          
      ...
        }

        static final int TRANSACTION_fibJR = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_fibJI = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_fibNR = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        static final int TRANSACTION_fibNI = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
        static final int TRANSACTION_fib = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
    }
    public long fibJR(long n) throws android.os.RemoteException;
    public long fibJI(long n) throws android.os.RemoteException;
    public com.zgq.android.fibonaccicommon.FibonacciResponse fib(com.zgq.android.fibonaccicommon.FibonacciRequest request) throws android.os.RemoteException;
}
```
然后是server端的实现，主要是继承Stub类，代码如下：
```java
package com.zgq.android.fibonacciservice;
//import com.zgq.android.f

import android.os.RemoteException;
import android.os.SystemClock;
import android.util.Log;

import com.zgq.android.fibonaccicommon.FibonacciRequest;
import com.zgq.android.fibonaccicommon.FibonacciResponse;
import com.zgq.android.fibonaccicommon.IFibonacciService;
import com.zgq.android.fibonaccinative.Fiblib;

/**
 * Created by zgq on 2018/3/18.
 */

public class IFibonacciServiceImpl extends IFibonacciService.Stub{

    public static final String TAG = "IFibonacciServiceImpl";
    @Override
    public long fibJR(long n) throws RemoteException {
        return Fiblib.fibJR(n);
    }

    @Override
    public long fibJI(long n) throws RemoteException {
        return Fiblib.fibJI(n);
    }

    @Override
    public FibonacciResponse fib(FibonacciRequest request) throws RemoteException {
        Log.d(TAG,String.format("fib(%d,%s)",request.getN(),request.getType()));
        long timeInMillis = SystemClock.uptimeMillis();
        long result;
        switch (request.getType()){
            case ITERATIVE_JAVA:
                result = this.fibJI(request.getN());
                break;
            case RECURSIVE_JAVA:
                result = this.fibJR(request.getN());
                break;
                default:
                    return null;
        }
        timeInMillis = SystemClock.uptimeMillis()-timeInMillis;

        return new FibonacciResponse(result,timeInMillis);
    }
}
```
然后需要将创建好的FibnacciserviceImpl绑定到android service组件上，供client端调用。代码如下：
```java
package com.zgq.android.fibonacciservice;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.util.Log;

import com.zgq.android.fibonaccicommon.IFibonacciService;

/**
 * Created by zgq on 2018/3/18.
 */

public class FibonacciService extends Service {
    private static final String TAG = "FibonacciService";
    private IFibonacciServiceImpl service;

    @Override
    public void onCreate() {
        super.onCreate();
        this.service = new IFibonacciServiceImpl();
        Log.d(TAG,"onCreate");
    }
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG,"onBind");
        return this.service;
    }
    @Override
    public boolean onUnbind(Intent intent) {
        return super.onUnbind(intent);
    }
    @Override
    public void onDestroy() {
        this.service = null;
        super.onDestroy();
    }
}

```
最后是client端的实现,代码如下：
```java
package com.zgq.android.fibonacciclient;
...
import com.zgq.android.fibonaccicommon.FibonacciRequest;
import com.zgq.android.fibonaccicommon.FibonacciResponse;
import com.zgq.android.fibonaccicommon.IFibonacciService;
import com.zgq.android.fibonacciservice.FibonacciService;



public class FibonacciActivity extends Activity implements View.OnClickListener,ServiceConnection{

    private static final String TAG = "FibonacciActivity";
    private EditText input;
    private Button button;
    private RadioGroup type;
    private TextView outPut;
    private IFibonacciService service;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        this.input = super.findViewById(R.id.input);
        this.button = super.findViewById(R.id.button);
        this.outPut = super.findViewById( R.id.output);
        this.type = super.findViewById(R.id.type);
        this.button.setOnClickListener(this);
        this.button.setEnabled(false);

    }

    @Override
    protected void onResume() {
        Log.d(TAG,"onResume");
        super.onResume();
        Intent intent = new Intent(this,FibonacciService.class);
        intent.setAction(IFibonacciService.class.getName());
        if(!super.bindService(intent,this,BIND_AUTO_CREATE)){
            Log.w(TAG,"fail to bind service");
        }
    }

    @Override
    protected void onPause() {
        Log.d(TAG,"onPause()'ed");
        super.onPause();
        super.unbindService(this);
    }

    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        boolean isLocal = false;
        Log.d(TAG,"onServiceConnected()'ed"+componentName);
        IInterface ll = iBinder.queryLocalInterface("com.zgq.android.fibonaccicommon.IFibonacciService");
        if(ll!=null && ll instanceof com.zgq.android.fibonaccicommon.IFibonacciService)
            isLocal = true;
        Log.d(TAG,String.valueOf(isLocal));
        this.service = IFibonacciService.Stub.asInterface(iBinder);
        this.button.setEnabled(true);
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        Log.d(TAG,"onServiceDisconnected()'ed"+componentName);
        this.service = null;
        this.button.setEnabled(false);
    }

    @Override
    public void onClick(View view) {
        final long n;
        String s = this.input.getText().toString();
        if(TextUtils.isEmpty(s)){
            return;
        }
        try {
            n = Long.parseLong(s);
        }catch (NumberFormatException e){
            this.input.setError(super.getText(R.string.input_error));
            return;
        }
        final FibonacciRequest.Type type;
        switch (FibonacciActivity.this.type.getCheckedRadioButtonId()){
            case R.id.type_fib_jr:
                type = FibonacciRequest.Type.RECURSIVE_JAVA;
                break;
            case R.id.type_fib_ji:
                type = FibonacciRequest.Type.ITERATIVE_JAVA;
                break;
                default:
                    return;
        }
        final FibonacciRequest request = new FibonacciRequest(n,type);
        final ProgressDialog dialog = ProgressDialog.show(this,"",super.getText(R.string.progress_text),true);
        new AsyncTask<Void,Void,String>(){
            @Override
            protected String doInBackground(Void... voids) {
                try{
                    long totalTime = SystemClock.uptimeMillis();
                    FibonacciResponse response = FibonacciActivity.this.service.fib(request);
                    totalTime = SystemClock.uptimeMillis()-totalTime;
                    return String.format("fibonacci(%d)=%d\nin %d ms\n(+%d ms)",n,response.getResult(),response.getTimeInMullis(),totalTime-response.getTimeInMullis());
                }catch (RemoteException e){
                    Log.wtf(TAG,"fail to communicate with the service",e);
                    return  null;
                }
            }

            @Override
            protected void onPostExecute(String s) {
                dialog.dismiss();
                if(s == null){
                    Toast.makeText(FibonacciActivity.this,R.string.fib_error,Toast.LENGTH_SHORT).show();
                }else{
                    FibonacciActivity.this.outPut.setText(s);
                }
            }
        }.execute();

    }
}
```
