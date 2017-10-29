# 关于ServiceManager

通过Binder的通讯机制可知，客户端进程要实现与服务端进程通讯，关键在于获取服务端提供的Binder引用。一般情况下，我们是通过在客户端创建一个ServiceConnection，然后交给服务端进程，用来获取其Binder，而在Activity创建过程，获取各系统服务的Binder则是通过ServiceManager进行的。

ServiceManager本身也是一个Binder：

```
 //服务端进程接口
 public interface IServiceManager extends IInterface

 //对应AIDL生成的通讯类
 public abstract class ServiceManagerNative extends Binder implements IServiceManager
```

当我们第一次请求获取ServiceManager时，会通过ServiceManagerNative 的asInterface创建对应的Proxy类：  
 ServiceManagerProxy;

```
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
```

**注意：**此处的服务端Binder是通过BinderInternal.getContextObject\(\)进行获取，对应的Binder_可能是_从C++层获取到的；

```
 /**
 * Return the global "context object" of the system.  This is usually
 * an implementation of IServiceManager, which you can use to find
 * other services.
 */
public static final native IBinder getContextObject();
```

需要知道的是，ServiceManager虽然自身就是一个Binder，但是其中包含所有系统能够提供服务的Binder引用！

当我们需要获取对应的系统的服务时，首先需要通过ServiceManager来获取对应服务的Binder，而这一步，ServiceManager是通过远程通讯，向底层系统进程请求的；

```
 public static IBinder getService(String name) {
    try {
        //如果本地已经缓存了对应的Binder引用，则直接从缓存中获取
        IBinder service = sCache.get(name);
        if (service != null) {
            return service;
        } else {
            //通过进程通讯，请求系统进程创建
            return getIServiceManager().getService(name);
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
```

请求系统返回对应系统服务的Binder引用给ServiceManager：

```
 public IBinder getService(String name) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
    IBinder binder = reply.readStrongBinder();
    reply.recycle();
    data.recycle();
    return binder;
}
```

# 其他系统服务缓存到ServiceManager

其他的系统服务，在创建自己的Binder引用之后，会主动将自己注册到ServiceManager的远程服务中进行缓存，通过ServiceManager的addService方法：

```
 /**
 * Place a new @a service called @a name into the service
 * manager.
 * 
 * @param name the name of the new service
 * @param service the service object
 */
public static void addService(String name, IBinder service) {
    try {
        getIServiceManager().addService(name, service, false);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}

//最终通过代理类ServiceManagerProxy添加到远程服务中进行缓存
public void addService(String name, IBinder service, boolean allowIsolated)
        throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    data.writeStrongBinder(service);
    data.writeInt(allowIsolated ? 1 : 0);
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    reply.recycle();
    data.recycle();
}
```

个人猜测这样做的目的是在应用启动时，一次性从ServiceManager的远程服务中拿出所有缓存的服务Binder引用，后面需要时直接从缓存拿，无需再进行创建：

在ActivityThread的bindApplication方法中：

```
  public final void bindApplication(String processName, ApplicationInfo appInfo,
        List<ProviderInfo> providers, ComponentName instrumentationName,
        ProfilerInfo profilerInfo, Bundle instrumentationArgs,
        IInstrumentationWatcher instrumentationWatcher,
        IUiAutomationConnection instrumentationUiConnection, int debugMode,
        boolean enableBinderTracking, boolean trackAllocation,
        boolean isRestrictedBackupMode, boolean persistent, Configuration config,
        CompatibilityInfo compatInfo, Map<String, IBinder> services, Bundle coreSettings) {

    if (services != null) {
        // Setup the service cache in the ServiceManager
        ServiceManager.initServiceCache(services);
    }


 /**
 * This is only intended to be called when the process is first being brought
 * up and bound by the activity manager. There is only one thread in the process
 * at that time, so no locking is done.
 * 
 * @param cache the cache of service references
 * @hide
 */
public static void initServiceCache(Map<String, IBinder> cache) {
    if (sCache.size() != 0) {
        throw new IllegalStateException("setServiceCache may only be called once");
    }
    sCache.putAll(cache);
}
```



