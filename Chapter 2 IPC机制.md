





# Chapter 2 IPC机制#

##  2.1 Android IPC 简介

- IPC 是Inter-Process Communication 的缩写，为进程间通信或者跨进程通信，两个进程间进行交换数据的过程。

- 线程 vs 进程

  - 线程是cpu调度的最小单元，同时线程是一种有限的系统资源；
  - 进程一般指一个执行单元，一个程序或应用可以包括很多线程。

- 多进程两个情况：

  - 自身采用多进程模式
  - 当前应用向其他应用获取数据

## 2.2 Android 中的多进程模式

- Android中使用多进程只有一种方法，四大组件（Activity，Service, Receiver, Content Provider) 在AndroidManifest 中指定 android:process属性。
- android:process属性中   ”：“ vs ”.“
  - ":" 要在当前进程名前附加上当前包名，简写
  - ”.“ 完整的命名方式，eg: "com.ryg.chapter_2.remote"
  - ":" 属于当前应用的私有进程，其他应用组件不可以跑在同一进程中
  - "." 全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中
- 多进程造成的问题：
  - 静态成员和单例模式完全失效。这是因为Android为每一个进程分配独立的虚拟机，不同的虚拟机有不同的内存分配上有不同的地址空间，不同的虚拟机中访问的同一个类对象会产生很多副本。
  - 线程同步机制完全失效。
  - SharedPreferences 的可靠性下降。
  - Application会多次创建。

## 2.3 IPC基础概念介绍

### 2.3.1 Serializable 接口

- 是java所提供的一个序列化接口，是一个空接口，为对象提供标准的序列化和反序列化操作。

- 想要一个对象实现序列化，只需这个类实现Serializable接口并声明serialVersionUID即可。 For example:

  ```java
  public class User implements Serializable{
    private static final long serialVersionUID= 51906712372195773L;
    
    public int userId;
    public String userName;
    public boolean isMale;
    ...
  }
  ```

- 对象的序列化和反序列化也非常简单，只需采用 `ObjectOutputStream`  和 `ObjectInputStream` 。 For example:

  ```java
  // 序列化过程
  User user=new User(0,"jake",true);
  ObjectOutputStream out= new ObjectOutputStream(new FileOutputStream("cache.txt"));
  out.writeObject(user);
  out.close();

  //反序列化过程
  ObjectInputStream in= new ObjectInputStream(new FileInputStream("cache.txt"));
  User newUser=(user) in.readObject();
  in.close();
  ```

- `serialVersionUID` 是用来辅助序列化和反序列化的过程的。 只有序列化后的数据中`serialVersionUID` 和当前类的`serialVersionUID` 相同才能正常的反序列化。 序列化时系统会先把当前类的`serialVersionUID` 写入序列化的文件中，反序列化的时候，系统会去检测文件中的`serialVersionUID` 是否和当前类的一致。

- 静态成员变量属于类不属于对象，不会参与序列化过程。 transient 关键字标记的成员变量不参与序列化过程

### 2.3.2 `Parcelable` 接口 

- `Parcelable` 也是一种接口，实现这个接口，一个类的对象就可以实现序列化并可以通过`Intent` 和`Binder` 传递。 For example:

  ```java
  public class user implements Parcelable{
    public int userId;
    public String userName;
    public boolean isMale;
    
    public Book book;
    
    public user(int userId,String userName,boolean isMale)
    {
      this.userId=userId;
      this.userName=userName;
      this.isMale=isMale;
    }
    //返回当前对象的内容描述，如果含有文件描述，返回1，否则返回0. 几乎所有情况都返回0；
    public int describleContents(){ return 0;}
    
    public void writeToParcel(Parcel out,int flags)
    {
      out.writeInt(userId);
      out.writeString(userName);
      out.writeInt(isMale ? 1:0);
      out.writeParcelable(book,0);
    }
    
    public static final Parcelable.Creator<user> CREATOR= new Parcelable.Creator<user>(){
      public user createFromParcel(Parcel in)
      {
        return new user(in);
      }
      public user[] newArray(int size)
      {
        return new user[size];
      }
    };
    
    private user(Parcel in)
    {
      userId=in.readInt();
      userName=in.readString();
      isMale = in.readInt()==1;
      book=in.readParcelable(Thread.currentThread().getContextClassLoader());
    }
  }
  ```

- `Parcel` 内部包装了可序列化的数据，可以在Binder中自由传输。book是一个可序列化对象，所以它的反序列化过程需要传递线程的上下文类加载器

- Parcelable主要用在内存序列化上，存储设备中或者对象序列化后通过网络传输建议用Serializable。 

- Serializable 用起来简单，开销很大，序列化和反序列化过程需要大量I/O 操作，会产生大量的临时变量容易触发GC，使用了反射所以效率低；Parcelable用起来麻烦，效率高。

### 2.3.3 Binder

- 不同角度的Binder：
  - 直观： Android的一个类，实现了IBinder接口
  - IPC角度： Android中的一种跨进程通信方式
  - 虚拟的物理设备，设备驱动是/dev/binder
  - Android Framework 角度： ServiceManager 连接各种Manager（ActivityManager、WindowManager，等）和相应的ManagerService的桥梁
  - 应用层：客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器端提供的服务或者数据，服务包括普通服务和基于AIDL的服务
####2.3.3.1 基于AIDL实例所生成的Binder类


  - Book.java 表示图书信息的类，实现了Parcelable接口

  - Book.aidl  Book类在AIDL中的申明。

    ```java
     package com.ryg.chapter_2.aidl;
     parcelable Book;
    ```

  - IBookManager.aidl  定义了一个接口，里面有两个方法： ` getBookList` 和 `addBook` 。尽管Book类和IBookManager位于相同的包中，但是在IBookManager中仍要导入Book类。

  - 根据系统生成的Binder类来分析Binder的工作原理

    - 系统生成的IBookManager.java 这个类，他继承了 `IInterface` 这个接口，同时自己也是接口。

    - 首先声明了两个方法 ` getBookList` 和 `addBook` ，同时申明了两个整型的id分别用于标志这两个方法，用于在transact过程中客户端所请求的方法。

    - 声明了一个内部类 `Stub` ， 这就是一个Binder类，当客户端与服务端都位于同一个进程时，方法调用不会走跨进程的`transact` 过程；位于不同进程时，调用`transact` 过程。这个逻辑有`Stub`的内部代理类`Proxy`完成。
#####2.3.3.2 `Stub` 类和 其内部代理类`Proxy` 方法的含义 

- `DESCRIPTOR` ： `Binder` 的唯一标识，一般用当前的`Binder` 的类名表示。

- `asInterface(android.os.IBinder obj)` ：用于将服务端的`Binder` 对象转换成客户端所需的AIDL接口类型的对象；若客户端与服务端位于同一进程，那么此方法返回的是服务端的`Stub` 对象本身，否则返回的是系统封装后的`Stub.proxy` 对象。

- `asBinder` ： 返回当`Binder` 对象。

- `onTransact` ：运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法处理。

  ```java
  public Boolean onTransact(int code,android.os.Parcel data, android.os.Parcel reply, int flags)
  ```

  1. 服务端通过code确定客户端请求的方法；
  2. 从data中取出目标方法所需的参数；
  3. 执行目标方法，向reply中写入返回值；
  4. 如果此方法返回false，那么客户端的请求失败，因此可以用此特性来做权限验证。

- `Proy#getBookList`： 运行在客户端

  1. 创建所需要的输入型`Parcel` 对象`_data` 、输出型`Parcel`对象`_reply`和返回值对象`list`；
  2. 参数信息写入`_data` 中；
  3. 调用`transact`方法来发起RPC（远程过程调用）请求，同时当前线程挂起；
  4. 服务端`onTransact`方法会被调用，知道RPC过程返回后，当前线程继续执行，并从`_reply` 中取出RPC过程的返回结果；
  5. 返回`_reply`中的数据。

- 客户端发起远程请求时，当前线程会被挂起直至服务端进程返回数据，如果远程方法是一个耗时的，不能在UI线程中发起此远程请求；

- 服务端的Binder方法运行在Binder的线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去实现，因为他已经运行在一个线程中了。

![Binder机制](https://raw.githubusercontent.com/JingxuanChen/Android-----/master/binder%E6%9C%BA%E5%88%B6.png)

#####2.3.3.3 `linkToDeath` 和 `unlinkToDeath` 

- 通过`linkToDeath` 设置一个死亡代理，当Binder死亡时，我们能收到通知。

  1. 声明一个`DeathRecipient` 对象，有一个`DeathRecipient`是一个接口，其内部只有一个方法 `binderDied`， 当Binder死亡时，系统就会回调`binderDied` 方法。

     ```java
     private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient(){
       @Override
       public void binderDied(){
         if(mBookManager==null)
           return;
         		
       mBookManager.asBinder().unlinkToDeath(mDeathRecipient,0);
         mBookManager=null;
         //TODO:重新绑定远程服务
       }
     };
     ```

  2.  在客户端绑定远程服务成功后，给binder设置死亡代理：

     ```java
     mService = IBookManager.Stub.asInterface(binder);
      binder.linkToDeath(mDeathRecipient,0);
     ```

  3.  通过binder的`isBinderAlive` 也可以判断Binder是否死亡

### 2.4 Android中的IPC方式

#### 2.4.1 使用Bundle

四大组件中的三大组件（Activity、Service、Receiver）都支持在Intent中传递Bundle数据。传输的数据必须能够被序列化，比如基本类型、实现Parcelable接口的对象、实现了Serializable接口的对象和一些Android支持的特殊对象。

A进程正在进行一个计算，计算完成后要启动B进程的一个组件并把计算结果传递给B，但是计算结果无法放入Bundle----解决办法，通过Intent启动进程B的一个Service组件，让service组件在后台计算，计算后再启动B进程的组件。 核心思想：将原本需要在A进程的计算任务转移到B进程的后台Service中执行。

#### 2.4.2 使用文件共享

两个进程通过读/写同一个文件来 交换数据。还可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象。文件格式没有要求。

局限性：并发读/写的问题，读出的内容可能不是最新的，如果并发写那更严重。避免并发写或者使用线程同步限制多线程写操作。

SharedPreferences是Android提供的轻量级存储方案，通过键值对方式存储数据。系统对它的读写有一定的缓存策略，即内存中会有一份SharedPreferences文件的缓存，多进程读/写变得不可靠。

#### 2.4.3 使用Messenger（信使）

- 在不同的进程中传递Message对象，在Message中放入需要传递的数据。轻量级IPC方案，底层实现是AIDL。

- 构造方法：

  ```java
  public Messenger(Handler target)
  {
    mTarget=target.getIMessager();
  }

  public Messenger(IBinder target)
  {
    mTarget=IMessenger.Stub.asInterface(target);
  }
  ```

  服务端一次处理一个请求，不用考虑线程同步的问题。

- 服务端进程：

  - 创建一个Service处理客户端的连接请求；
  - 创建一个Handler并通过它来创建一个Messenger对象；
  - 在Service的onBind中返回Messenger对象底层的Binder即可。

- 客户端进程：

  - 绑定服务端的Service；

  - 用服务端返回的IBinder对象创建一个Messenger对象，通过这个Messenger就可以向服务端发送消息，类型为Message对象。

  - 若需要服务端能够回应客户端，创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。
    ![](https://raw.githubusercontent.com/JingxuanChen/Android-----/master/messenger%E6%9C%BA%E5%88%B6.png)


####2.4.4 使用AIDL

- 服务端：

  - 创建一个service来监听客户端的连接请求
  - 创建一个AIDL文件，将暴露给客户端的接口在AIDL文件中声明
  - 在Service中实现AIDL接口

- 客户端：

  - 绑定服务端service
  - 服务端返回的Binder对象转换成AIDL接口所属的类型
  - 调用AIDL中的方法

- AIDL接口的创建：

  - 在AIDL文件中能用的数据类型：
    - 基本数据类型（int, long, char, boolean, double等）；
    - String 和 CharSequence;
    - List: 只支持ArrayList, 里面的每个元素都必须能够被AIDL支持；
    - Map：只支持HashMap, 里面的每个元素都必须被AIDL支持，包括key和value;
    - Parcelable: 所有实现了Parcelable接口的对象；
    - AIDL: 所有的AIDL接口本身也可以在AIDL文件中使用。
  - 其中自定义的Parcelable对象和AIDL对象必须要显示import进来，无论它们是否和当前的AIDL文件位于同一个包内。
  - 如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和他同名的AIDL文件，并在其中声明他为Prcelable对象。
  - 除了基本数据类型，其他类型的参数必须标上方向：in, out, inout；
  - AIDL接口只支持方法，不支持声明静态常量；

  ```java
  //IBookManager.aidl
  package com.ryg.chapter_2.aidl;

  import com.ryg.chapter_2.aidl.Book;

  Interface IBookManager{
    List<Book> getBookList();
    void addBook(in Book book);
  }

  //Book.aidl
  package com.ryg.chapter_2.aidl
  parcelable Book;
  ```

- 远程服务端Service的实现

  ```java
  public class BookManagerService extends Service{
    
    private static final String TAG="BMS";
    
    private CopyOnWriteArrayList<Book> mBookList=new CopyOnWriteArrayList<Book>();
    
    private Binder mBinder=new IBookManager.Stub(){
      @Override
      public List<Book> getBookList() throws RemoteException{
        return mBookList;
      }
      
      @Override
      public void addBook(Book book) throws RemoteException{
        mBookList.add(book)''
      }
    };
     
    @Override
    public void onCreate(){
      super.onCreatre();
      mBookList.add(new Book(1,"Android"));
      mBookList.add(new Book(2,"Ios"));
    }
    
    @Override
    public IBinder onBind(Intent intent)
    {
      return mBinder;
    }
  }
  ```

  - CopyOnWriteArrayList 支持并发读/写；AIDL方法在服务端的Binder线程池中执行，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形，所以要在AIDL方法中处理线程同步。
  - 在XML中注册这个Service，运行在独立的进程。

- 客户端的实现：

  ```java
  public class BookManagerActivity extends Activity{
    private static final String TAG="BookManagerActivity";
    
        private ServiceConnection mConnection = new ServiceConnection() {
          public void onServiceConnected(ComponentName className, IBinder service) {
              IBookManager bookManager = IBookManager.Stub.asInterface(service);
              mRemoteBookManager = bookManager;
              try {
                  List<Book> list = bookManager.getBookList();
                  Log.i(TAG, "query book list, list type:"
                          + list.getClass().getCanonicalName());
                  Log.i(TAG, "query book list:" + list.toString());
                  Book newBook = new Book(3, "Android进阶");
                  bookManager.addBook(newBook);
                  Log.i(TAG, "add book:" + newBook);
                  List<Book> newList = bookManager.getBookList();
                  Log.i(TAG, "query book list:" + newList.toString());
                  bookManager.registerListener(mOnNewBookArrivedListener);
              } catch (RemoteException e) {
                  e.printStackTrace();
              }
          }

          public void onServiceDisconnected(ComponentName className) {
              Log.d(TAG, "onServiceDisconnected. tname:" + Thread.currentThread().getName());
          }
      };
    
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_book_manager);
          Intent intent = new Intent(this, BookManagerService.class);
          bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
      }
  }
  ```

- 解注册功能：使用RemoteCallbackList

  - RemoteCallbackList 是系统专门提供的用于删除跨进程listener的接口。 是一个泛型，支持管理任意的AIDL接口。

  - 工作原理： 在它的内部有一个Map结构，用来保存所有的AIDL回调，key是IBinder类型，value是Callback类型, Callback中封装了真正的远程listener。

    ```java
    ArrayMap<IBinder，Callback> mCallbacks=new ArrayMap<IBinder,Callback>();
    ```

  - 当客户端注册listener的时候，会把listener的信息存入mCallbacks

    ```java
    IBinder key=listener.asBinder();
    Callback value=new Callback(listner, cookie);
    ```

#### 2.4.5 使用ContentProvider

- 底层实现同样是Binder。

- 自定义ContentProvider类继承ContentProvider类并实现六个抽象方法：onCreate、query、update、insert、delete和getType。 getType用来返回一个Uri请求所对应的MIME类型（媒体类型），比如图片、视频等，若不关心，则返回null。六个方法运行在ContentProvider的进程，除了onCreate由系统回调并运行在主线程里，其他的五个方法均运行有外界回调并运行在Binder线程池中。

- ContentProvider主要以表格的形式来组织数据，并且可以包含多个表。对每个表来说，他们具有行、列的层次性。行对应一条记录，列对应一条记录的一个字段。

- ContentProvider还支持文件数据。处理时，返回文件的句柄给外界从而让文件来访问ContentProvider的文件信息。

- 注册自定义的ContentProvider---BookProvider

  ```xml
  <provider
  	android:name=".provider.BookProvider"
       android:authorities="com.ryg.chapeter_2.book.provider"
       android:permission="com.ryg.PROVIDER"
       android:process=":provider">
  </provider>
  ```

  - android:authorities 是ContentProvider的唯一标识，通过这个属性外部应用就能访问我们的 BookProvider, 因此 android：authorities必须是唯一的。
  - 为了进程间通信，让次BookProvider运行在独立的进程中并给他添加权限，外界应用如果想访问BookProvider，就必须声明“com.ryg.PROVIDER” 这个权限。ContentProvider的权限还可以细分为读权限和写权限，分别对应 android:readPermission 和 android:writePermission属性。

#### 2.4.6 使用 Socket

- 需要声明权限

  ```xml
  <user-permission android:name="android.permission.INTERNET" />
  <user-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
  ```

- 其次注意不能在主线程访问网络，在Android4.0 及以上的设备中运行，会抛出异常：android.os.NetworkOnMainThreadException。 

### 2.5 Binder 连接池

- 当AIDL变多时，Service也变多，需要减少Service的数量，将所有的AIDL放在同一个Service中管理。

- 工作机制：每个业务创建自己的AIDL接口并实现此接口，这个时候不同的业务模块之间是不能耦合的，所有的实现细节我们单独，然后向服务端提供自己唯一标识和其对应的Binder对象；

- 对于服务端，只需要一个Service，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征返回相应的Binder对象，不同的业务模块拿到所需的Binder对象后，就可以进行远程方法调用。

- Binder连接池主要作用将每个业务模块的Binder请求同一转发到远程Service中执行，从而避免重复创建Service的过程。

  ![](https://raw.githubusercontent.com/JingxuanChen/Android-----/master/Binder%E8%BF%9E%E6%8E%A5%E6%B1%A0%E6%9C%BA%E5%88%B6.png)

- 例子： 两个AIDL接口（ISecurityCenter 和 ICompute）

  - 声明两个AIDL接口在两个不同的文件。

  - 实现两个AIDL接口，SecurityCenterImpl extends ISecurityCenter.Stub， ComputerImpl extends ICompute.Stub 两个java文件

  - 声明Binder连接池创建AIDL接口 IBinderPool.aidl

    ```java
    package com.ryg.chapter_2.binderpool;

    interface IBinderPool {

        /**
         * @param binderCode, the unique token of specific Binder<br/>
         * @return specific Binder who's token is binderCode.
         */
        IBinder queryBinder(int binderCode);
    }
    ```

  - 为Binder连接池创建远程Service并实现IBinderPool

    ```java
    package com.ryg.chapter_2.binderpool;

    import java.util.concurrent.CountDownLatch;

    import android.content.ComponentName;
    import android.content.Context;
    import android.content.Intent;
    import android.content.ServiceConnection;
    import android.os.IBinder;
    import android.os.RemoteException;
    import android.util.Log;

    public class BinderPool {
        private static final String TAG = "BinderPool";
        public static final int BINDER_NONE = -1;
        public static final int BINDER_COMPUTE = 0;
        public static final int BINDER_SECURITY_CENTER = 1;

        private Context mContext;
        private IBinderPool mBinderPool;
        private static volatile BinderPool sInstance;
        private CountDownLatch mConnectBinderPoolCountDownLatch;

        private BinderPool(Context context) {
            mContext = context.getApplicationContext();
            connectBinderPoolService();
        }

        public static BinderPool getInsance(Context context) {
            if (sInstance == null) {
                synchronized (BinderPool.class) {
                    if (sInstance == null) {
                        sInstance = new BinderPool(context);
                    }
                }
            }
            return sInstance;
        }

        private synchronized void connectBinderPoolService() {
            mConnectBinderPoolCountDownLatch = new CountDownLatch(1);
            Intent service = new Intent(mContext, BinderPoolService.class);
            mContext.bindService(service, mBinderPoolConnection,
                    Context.BIND_AUTO_CREATE);
            try {
                mConnectBinderPoolCountDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        /**
         * query binder by binderCode from binder pool
         * 
         * @param binderCode
         *            the unique token of binder
         * @return binder who's token is binderCode<br>
         *         return null when not found or BinderPoolService died.
         */
        public IBinder queryBinder(int binderCode) {
            IBinder binder = null;
            try {
                if (mBinderPool != null) {
                    binder = mBinderPool.queryBinder(binderCode);
                }
            } catch (RemoteException e) {
                e.printStackTrace();
            }
            return binder;
        }

        private ServiceConnection mBinderPoolConnection = new ServiceConnection() {

            @Override
            public void onServiceDisconnected(ComponentName name) {
                // ignored.
            }

            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                mBinderPool = IBinderPool.Stub.asInterface(service);
                try {
                    mBinderPool.asBinder().linkToDeath(mBinderPoolDeathRecipient, 0);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                mConnectBinderPoolCountDownLatch.countDown();
            }
        };

        private IBinder.DeathRecipient mBinderPoolDeathRecipient = new IBinder.DeathRecipient() {
            @Override
            public void binderDied() {
                Log.w(TAG, "binder died.");
                mBinderPool.asBinder().unlinkToDeath(mBinderPoolDeathRecipient, 0);
                mBinderPool = null;
                connectBinderPoolService();
            }
        };

        public static class BinderPoolImpl extends IBinderPool.Stub {

            public BinderPoolImpl() {
                super();
            }

            @Override
            public IBinder queryBinder(int binderCode) throws RemoteException {
                IBinder binder = null;
                switch (binderCode) {
                case BINDER_SECURITY_CENTER: {
                    binder = new SecurityCenterImpl();
                    break;
                }
                case BINDER_COMPUTE: {
                    binder = new ComputeImpl();
                    break;
                }
                default:
                    break;
                }

                return binder;
            }
        }

    }
    ```

    BinderPool是一个单例模式。

  - 在线程种执行验证：

    ```java
    private void doWork() {
            BinderPool binderPool = BinderPool.getInsance(BinderPoolActivity.this);
            IBinder securityBinder = binderPool
                    .queryBinder(BinderPool.BINDER_SECURITY_CENTER);
            
            mSecurityCenter = (ISecurityCenter) SecurityCenterImpl
                    .asInterface(securityBinder);
            Log.d(TAG, "visit ISecurityCenter");
            String msg = "helloworld-安卓";
            System.out.println("content:" + msg);
            try {
                String password = mSecurityCenter.encrypt(msg);
                System.out.println("encrypt:" + password);
                System.out.println("decrypt:" + mSecurityCenter.decrypt(password));
            } catch (RemoteException e) {
                e.printStackTrace();
            }

            Log.d(TAG, "visit ICompute");
            IBinder computeBinder = binderPool
                    .queryBinder(BinderPool.BINDER_COMPUTE);
            ;
            mCompute = ComputeImpl.asInterface(computeBinder);
            try {
                System.out.println("3+5=" + mCompute.add(3, 5));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    ```

    通过CountDownLatch将 bindService异步操作转换成了同步操作。

### 2.6 选用合适的IPC方式

![](https://raw.githubusercontent.com/JingxuanChen/Android-----/master/%E9%80%89%E7%94%A8%E5%90%88%E9%80%82%E7%9A%84ipc.png)

