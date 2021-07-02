# 注册中心
[注册中心——开篇](https://segmentfault.com/a/1190000016905715)  


## 开篇
### 注册中心是什么？
服务治理框架中可以大致分为服务通信和服务管理两个部分，服务管理可以分为服务注册、服务发现以及服务被热加工介入，
服务提供者Provider会往注册中心注册服务，而消费者Consumer会从注册中心中订阅相关的服务，并不会订阅全部的服务。

#### （一）RegistryService
该接口是注册中心模块的服务接口，提供了注册、取消注册、订阅、取消订阅以及查询符合条件的已注册数据。



在上篇文章中就说明了dubbo是以总线模式来时刻传递和保存配置信息的，也就是配置信息都被放在URL上进行传递，随时可以取得相关配置信息，
而这里提到了URL有别的作用，就是作为类似于节点的作用，首先服务提供者（Provider）启动时需要提供服务，就会向注册中心写下自己的URL地址。
然后消费者启动时需要去订阅该服务，则会订阅Provider注册的地址，并且消费者也会写下自己的URL。

```
public interface RegistryService {

    //注册。 允许URI相同但参数不同的URL并存，不能覆盖，也就是说url值必须唯一的，不能有一模一样
    void register(URL url);

    //取消注册
    void unregister(URL url);

    //订阅，这里不是根据全URL匹配订阅的，而是根据条件去订阅，也就是说可以订阅多个服务。listener是用来监听处理注册数据变更的事件。
    void subscribe(URL url, NotifyListener listener);

    //取消订阅
    void unsubscribe(URL url, NotifyListener listener);

    //查询注册列表，通过url进行条件查询所匹配的所有URL集合
    List<URL> lookup(URL url);

}
```

#### （二）Registry
注册中心接口，该接口很好理解，就是把节点以及注册中心服务的方法整合在了这个接口里面。
```
public interface Registry extends Node, RegistryService {
}
```
可以看到该接口并没有自己的方法，就是继承了Node和RegistryService接口。这里的Node是节点的接口，里面协定了关于节点的一些操作方法
```
public interface Node {
    URL getUrl();
    boolean isAvailable();
    void destroy();
}
```

#### （三）RegistryFactory
```
@SPI("dubbo")
public interface RegistryFactory {
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```
该接口是一个可扩展接口，可以看到该接口上有个@SPI注解，并且默认值为dubbo，也就是默认扩展的是DubboRegistryFactory，
并且可以在getRegistry方法上可以看到有@Adaptive注解，那么该接口会动态生成一个适配器RegistryFactory$Adaptive，
并且会去首先扩展url.protocol的值对应的实现类

#### （四）NotifyListener
```
public interface NotifyListener {
    void notify(List<URL> urls);
}
```
该接口只有一个notify方法，通知监听器。当收到服务变更通知时触发

#### （五）support包下的AbstractRegistry
AbstractRegistry实现的是Registry接口，是Registry的抽象类。
为了减轻注册中心的压力，在该类中实现了把本地URL缓存到property文件中的机制，并且实现了注册中心的注册、订阅等方法。

1. 属性
```
    // URL的地址分隔符，在缓存文件中使用，服务提供者的URL分隔
    private static final char URL_SEPARATOR = ' ';
    // URL地址分隔正则表达式，用于解析文件缓存中服务提供者URL列表
    private static final String URL_SPLIT = "\\s+";
    // 日志输出
    protected final Logger logger = LoggerFactory.getLogger(getClass());
    // 本地磁盘缓存，有一个特殊的key值为registies，记录的是注册中心列表，其他记录的都是服务提供者列表
    private final Properties properties = new Properties();
    // 缓存写入执行器
    private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));
    // 是否同步保存文件标志
    private final boolean syncSaveFile;
    //数据版本号
    private final AtomicLong lastCacheChanged = new AtomicLong();
    // 已注册URL集合， 注册的URL不仅仅可以是服务提供者的，也可以是服务消费者的
    private final Set<URL> registered = new ConcurrentHashSet<URL>();
    // 订阅URL的监听器集合
    private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<>();
    // 某个消费者被通知的某一类型的URL集合
    // 第一个key是消费者的URL，对应的就是哪个消费者。value是一个map集合，该map集合的key是分类的意思，例如providers、routes等，value就是被通知的URL集合
    private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<>();
    // 注册中心 URL
    private URL registryUrl;
    // 本地磁盘缓存文件，缓存注册中心的数据
    private File file;
```
从上面可以看到除了注册中心相关的一些属性外，可以看到好几个是个属性跟磁盘缓存文件和读写文件有关的，
这就是上面提到的把URL缓存到本地property的相关属性这里有几个需要关注的点：
1.1 properties：properties的数据跟本地文件的数据同步，当启动时，会从文件中读取数据到properties，而当properties中数据变化时，会写入到file。
而properties是一个key对应一个列表，比如说key就是消费者的url，而值就是服务提供者列表、路由规则列表、配置规则列表。就是类似属性notified的含义。
需要注意的是properties有一个特殊的key为registies，记录的是注册中心列表。
1.2 lastCacheChanged：因为每次写入file都是全部覆盖的写入，不是增量的去写入到文件，所以需要有这个版本号来避免老版本覆盖新版本。
1.3 notified：跟properties的区别是第一数据来源不是文件，而是从注册中心中读取，第二个notified根据分类把同一类的值做了聚合。

2. 构造方法AbstractRegistry
```
    public AbstractRegistry(URL url) {
        // 把url放到registryUrl中
        setUrl(url);
        // Start file save timer
        // 从url中读取是否同步保存文件的配置，如果没有值默认用异步保存文件
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
        // 获得file路径
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            //创建文件
            file = new File(filename);
            if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
                if (!file.getParentFile().mkdirs()) {
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        // 把文件里面的数据写入properties
        loadProperties();
        // 通知监听器，URL 变化结果
        notify(url.getBackupUrls());
    }
```

3. filterEmpty
```
    protected static List<URL> filterEmpty(URL url, List<URL> urls) {
        if (urls == null || urls.isEmpty()) {
            List<URL> result = new ArrayList<URL>(1);
            result.add(url.setProtocol(Constants.EMPTY_PROTOCOL));
            return result;
        }
        return urls;
    }
```
就是判断url集合是否为空，如果为空，则把url中key为empty的值加入到集合。该方法只有在notify方法中用到，为了防止通知的URL变化结果为空

4. doSaveProperties
该方法主要是将内存缓存properties中的数据存储到文件中，并且在里面做了版本号的控制，防止老的版本数据覆盖了新版本数据。

5. loadProperties
```
    private void loadProperties() {
        if (file != null && file.exists()) {
            InputStream in = null;
            try {
                in = new FileInputStream(file);
                // 把数据写入到内存缓存中
                properties.load(in);
                if (logger.isInfoEnabled()) {
                    logger.info("Load registry store file " + file + ", data: " + properties);
                }
            } catch (Throwable e) {
                logger.warn("Failed to load registry store file " + file, e);
            } finally {
                if (in != null) {
                    try {
                        in.close();
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
        }
    }
```
该方法就是加载本地磁盘缓存文件到内存缓存，也就是把文件里面的数据写入properties，可以对比doSaveProperties方法，
其中关键的实现就是properties.load和properties.store的区别，逻辑并不难。

6. getCacheUrls
```
    public List<URL> getCacheUrls(URL url) {
        for (Map.Entry<Object, Object> entry : properties.entrySet()) {
            // key为某个分类，例如服务提供者分类
            String key = (String) entry.getKey();
            // value为某个分类的列表，例如服务提供者列表
            String value = (String) entry.getValue();
            if (key != null && key.length() > 0 && key.equals(url.getServiceKey())
                    && (Character.isLetter(key.charAt(0)) || key.charAt(0) == '_')
                    && value != null && value.length() > 0) {
                //分割出列表的每个值
                String[] arr = value.trim().split(URL_SPLIT);
                List<URL> urls = new ArrayList<URL>();
                for (String u : arr) {
                    urls.add(URL.valueOf(u));
                }
                return urls;
            }
        }
        return null;
    }
```
该方法是获得内存缓存properties中相关value，并且返回为一个集合，从该方法中可以很清楚的看出properties中是存储的什么数据格式

7. lookup
```
    @Override
    public List<URL> lookup(URL url) {
        List<URL> result = new ArrayList<URL>();
        // 获得该消费者url订阅的 所有被通知的 服务URL集合
        Map<String, List<URL>> notifiedUrls = getNotified().get(url);
        // 判断该消费者是否订阅服务
        if (notifiedUrls != null && notifiedUrls.size() > 0) {
            for (List<URL> urls : notifiedUrls.values()) {
                for (URL u : urls) {
                    // 判断协议是否为空
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        // 添加 该消费者订阅的服务URL
                        result.add(u);
                    }
                }
            }
        } else {
            // 原子类 避免在获取注册在注册中心的服务url时能够保证是最新的url集合
            final AtomicReference<List<URL>> reference = new AtomicReference<List<URL>>();
            // 通知监听器。当收到服务变更通知时触发
            NotifyListener listener = new NotifyListener() {
                @Override
                public void notify(List<URL> urls) {
                    reference.set(urls);
                }
            };
            // 订阅服务，就是消费者url订阅已经 注册在注册中心的服务（也就是添加该服务的监听器）
            subscribe(url, listener); // Subscribe logic guarantees the first notify to return
            List<URL> urls = reference.get();
            if (urls != null && !urls.isEmpty()) {
                for (URL u : urls) {
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        result.add(u);
                    }
                }
            }
        }
        return result;
    }
```
该方法是实现了RegistryService接口的方法，作用是获得消费者url订阅的服务URL列表
1. URL可能是消费者URL，也可能是注册在注册中心的服务URL，我在注释中在URL加了修饰，为了能更明白的区分。
2. 订阅了的服务URL一定是在注册中心中注册了的。

8. register && unregister
这两个方法实现了RegistryService接口的方法，里面的逻辑很简单。其中注册的逻辑就是把url加入到属性registered，
而取消注册的逻辑就是把url从该属性中移除。真正的实现是在FailbackRegistry类中

9. subscribe && unsubscribe
这两个方法实现了RegistryService接口的方法，分别是订阅和取消订阅
```
    @Override
    public void subscribe(URL url, NotifyListener listener) {
        if (url == null) {
            throw new IllegalArgumentException("subscribe url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("subscribe listener == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Subscribe: " + url);
        }
        // 获得该消费者url 已经订阅的服务 的监听器集合
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners == null) {
            subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = subscribed.get(url);
        }
        // 添加某个服务的监听器
        listeners.add(listener);
    }
```
从源代码可以看到，其实订阅也就是把服务通知监听器加入到subscribed中，具体的实现也是在FailbackRegistry类中

10. recover
恢复方法，在注册中心断开，重连成功的时候，会恢复注册和订阅。
```
    protected void recover() throws Exception {
        // register
        //把内存缓存中的registered取出来遍历进行注册
        Set<URL> recoverRegistered = new HashSet<URL>(getRegistered());
        if (!recoverRegistered.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover register url " + recoverRegistered);
            }
            for (URL url : recoverRegistered) {
                register(url);
            }
        }
        // subscribe
        //把内存缓存中的subscribed取出来遍历进行订阅
        Map<URL, Set<NotifyListener>> recoverSubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
        if (!recoverSubscribed.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover subscribe url " + recoverSubscribed.keySet());
            }
            for (Map.Entry<URL, Set<NotifyListener>> entry : recoverSubscribed.entrySet()) {
                URL url = entry.getKey();
                for (NotifyListener listener : entry.getValue()) {
                    subscribe(url, listener);
                }
            }
        }
    }
```

11. notify
```
protected void notify(List<URL> urls) {
    if (urls == null || urls.isEmpty()) return;
    // 遍历订阅URL的监听器集合，通知他们
    for (Map.Entry<URL, Set<NotifyListener>> entry : getSubscribed().entrySet()) {
        URL url = entry.getKey();

        // 匹配
        if (!UrlUtils.isMatch(url, urls.get(0))) {
            continue;
        }
        // 遍历监听器集合，通知他们
        Set<NotifyListener> listeners = entry.getValue();
        if (listeners != null) {
            for (NotifyListener listener : listeners) {
                try {
                    notify(url, listener, filterEmpty(url, urls));
                } catch (Throwable t) {
                    logger.error("Failed to notify registry event, urls: " + urls + ", cause: " + t.getMessage(), t);
                }
            }
        }
    }
}
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
    if (url == null) {
        throw new IllegalArgumentException("notify url == null");
    }
    if (listener == null) {
        throw new IllegalArgumentException("notify listener == null");
    }
    if ((urls == null || urls.isEmpty())
            && !Constants.ANY_VALUE.equals(url.getServiceInterface())) {
        logger.warn("Ignore empty notify urls for subscribe url " + url);
        return;
    }
    if (logger.isInfoEnabled()) {
        logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
    }
    Map<String, List<URL>> result = new HashMap<String, List<URL>>();
    // 将urls进行分类
    for (URL u : urls) {
        if (UrlUtils.isMatch(url, u)) {
            // 按照url中key为category对应的值进行分类，如果没有该值，就找key为providers的值进行分类
            String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            List<URL> categoryList = result.get(category);
            if (categoryList == null) {
                categoryList = new ArrayList<URL>();
                // 分类结果放入result
                result.put(category, categoryList);
            }
            categoryList.add(u);
        }
    }
    if (result.size() == 0) {
        return;
    }
    // 获得某一个消费者被通知的url集合（通知的 URL 变化结果）
    Map<String, List<URL>> categoryNotified = notified.get(url);
    if (categoryNotified == null) {
        // 添加该消费者对应的url
        notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
        categoryNotified = notified.get(url);
    }
    // 处理通知监听器URL 变化结果
    for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
        String category = entry.getKey();
        List<URL> categoryList = entry.getValue();
        // 把分类标实和分类后的列表放入notified的value中
        // 覆盖到 `notified`
        // 当某个分类的数据为空时，会依然有 urls 。其中 `urls[0].protocol = empty` ，通过这样的方式，处理所有服务提供者为空的情况。
        categoryNotified.put(category, categoryList);
        // 保存到文件
        saveProperties(url);
        //通知监听器
        listener.notify(categoryList);
    }
}
```
notify方法是通知监听器，url的变化结果，不过变化的是全量数据，全量数据意思就是是以服务接口和数据类型为维度全量通知，
即不会通知一个服务的同类型的部分数据，用户不需要对比上一次通知结果。这里要注意几个重点：

11.1 发起订阅后，会获取全量数据，此时会调用notify方法。即Registry 获取到了全量数据
11.2 每次注册中心发生变更时会调用notify方法虽然变化是增量，调用这个方法的调用方，已经进行处理，传入的urls依然是全量的。
11.3 listener.notify，通知监听器，例如，有新的服务提供者启动时，被通知，创建新的 Invoker 对象。

12.saveProperties
```
private void saveProperties(URL url) {
    if (file == null) {
        return;
    }
    try {
        // 拼接url
        StringBuilder buf = new StringBuilder();
        Map<String, List<URL>> categoryNotified = notified.get(url);
        if (categoryNotified != null) {
            for (List<URL> us : categoryNotified.values()) {
                for (URL u : us) {
                    if (buf.length() > 0) {
                        buf.append(URL_SEPARATOR);
                    }
                    buf.append(u.toFullString());
                }
            }
        }
        // 设置到properties中
        properties.setProperty(url.getServiceKey(), buf.toString());
        // 增加版本号
        long version = lastCacheChanged.incrementAndGet();
        if (syncSaveFile) {
            // 将集合中的数据存储到文件中
            doSaveProperties(version);
        } else {
            //异步开启保存到文件
            registryCacheExecutor.execute(new SaveProperties(version));
        }
    } catch (Throwable t) {
        logger.warn(t.getMessage(), t);
    }
}
```
该方法是单个消费者url对应在notified中的数据，保存在到文件，而保存到文件的操作是调用了doSaveProperties方法，
该方法跟doSaveProperties的区别是doSaveProperties方法将properties数据全部覆盖性的保存到文件，
而saveProperties只是保存单个消费者url的数据。

13. destroy
 该方法在JVM关闭时调用，进行取消注册和订阅的操作。具体逻辑就是调用了unregister和unsubscribe方法

#### （六）support包下的FailbackRegistry
FailbackRegistry继承了AbstractRegistry，AbstractRegistry中的注册订阅等方法，实际上就是一些内存缓存的变化，
而真正的注册订阅的实现逻辑在FailbackRegistry实现，并且FailbackRegistry提供了失败重试的机制。






