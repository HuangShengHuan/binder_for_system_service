 # getSystemService的过程
 

  ContextImpl.java
 
    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
  SystemServiceRegistry.java
    
    /**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
    
  当系统服务还没有被创建时，会走createService方法：
     
     /**
     * Base interface for classes that fetch services.
     * These objects must only be created during static initialization.
     */
    static abstract interface ServiceFetcher<T> {
        T getService(ContextImpl ctx);
    }
    
    /**
     * Override this class when the system service constructor needs a
     * ContextImpl and should be cached and retained by that context.
     */
    static abstract class CachedServiceFetcher<T> implements ServiceFetcher<T> {
        private final int mCacheIndex;

        public CachedServiceFetcher() {
            mCacheIndex = sServiceCacheSize++;
        }

        @Override
        @SuppressWarnings("unchecked")
        public final T getService(ContextImpl ctx) {
            final Object[] cache = ctx.mServiceCache;
            synchronized (cache) {
                // Fetch or create the service.
                Object service = cache[mCacheIndex];
                if (service == null) {
                    service = createService(ctx);
                    cache[mCacheIndex] = service;
                }
                return (T)service;
            }
        }

        public abstract T createService(ContextImpl ctx);
    }
    
   在SystemServiceRegistry第一次被调用时，对应的serviceFetcher已经被通过registerService注册进来：
   
       static{
           ...
           registerService(Context.ACCOUNT_SERVICE, AccountManager.class,
                new CachedServiceFetcher<AccountManager>() {
            @Override
            public AccountManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ACCOUNT_SERVICE);
                IAccountManager service = IAccountManager.Stub.asInterface(b);
                return new AccountManager(ctx, service);
            }});
   
           registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});
            
    可以看到，并不是所有ServiceFetcher都通过获取IBinder的方式来获取对应的ServiceManager，比如ACTIVITY_SERVICE，因为在ActivityThread创建时，其已经被获取过了，所以无需再获取一次；
     
    
    
    