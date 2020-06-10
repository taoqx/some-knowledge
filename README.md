# some knowledge

## Android
1. Android性能优化方案（如内存优化、网络优化、布局优化、电量优化、业务优化）

   > 内存优化：
   >
   > - 监控可用内存和内存使用量
   >   - 使用内存分析器监控可用内存和内存使用量，发起垃圾回收事件，拍摄Java堆快照分析
   >   - 在Activity#onTrimMemory()回调中响应与内存相关的事件
   >   - 查询系统ActivityManager.MemoryInfo对象，包含设备相关的信息，如可用内存、总内存、内存阈值等
   >   - 对于占用内存比较大的对象尤其是Bitmap，用完之后及时释放
   > - 使用内存效率更高的代码结构
   >   - 谨慎使用服务，系统会将此服务的进程始终保持在运行状态，导致服务进程代价昂贵，因为一旦服务使用了某部分RAM，那么这部分RAM就不再可供其他进程使用。应该避免使用持久性服务，建议采用JobScheduler方案替代；如果必须使用某些服务，建议使用IntentService，它会在处理完intent后自动关闭。
   >   - 使用经过优化后的数据容器，常规的HashMap效率低下，建议使用Android优化过的数据容器，如SparseArray，它可以避免系统对键的自动装箱过程。
   >   - 避免内存抖动，内存抖动会导致大量的垃圾回收事件。垃圾回收通常不会影响应用性能，但系统在垃圾回收时无法处理其他事情，如果短时间发生大量垃圾回收事件，就可能会影响帧渲染导致画面卡顿。因此在for/while循环、View#onDraw等方法中不要创建临时对象，那样会快速消耗新生代内存区域，导致内存抖动。通过内存分析工具，寻找内存抖动较高的位置，进行修复。
   > - 移除会占用大量内存的资源和库：代码中的资源和库可能在我们不知情的情况下占用大量内存，因此缩减apk大小、移除冗余臃肿的库或资源也很有必要。
   >
   > 网络优化：
   >
   > - 通过Gzip压缩，减少传输的数据量大小，也就能减少流量消耗与传输数据
   >
   > - DNS解析需要向运营商Local DNS发起解析请求，有域名劫持、解析失败等风险，解析时间也长，通过，通过HttpDns替代DNS解析或直接用ip代替域名的方式优化
   > - 协议层优化：HTTP1.1版本引入了“持久连接”，多个请求被复用，无需重建TCP连接；HTTP2引入了“多工”、头信息压缩、服务器推送等特性
   > - 图片下载优化：采用WebP可大幅度节省流量，相对jpg图片流量节省25-35%，相对于png图片节省将近80%
   > - 合并请求、减少请求次数：对于统计、上报类的接口，可以先将数据保存在本地，设置JobScheduler任务统一上传
   > - 使用网络缓存，减少网络请求次数
   > - 数据预取：根据用户特性预先判断用户下一步行为，空闲时预先请求数据
   >
   > 布局优化：
   >
   > - 每隔16ms系统就会发送一个VSYNC信号触发界面渲染，如果下一个16ms内渲染完成那么画面就是流畅的，否则会有卡顿感，因此需要保证实际帧率不能低于60fps太多
   >
   > - 调试GPU过度渲染，精准定位过度渲染位置
   > - 使用扁平化的布局，移除不必要的View
   > - 使用Merge标签配合include减少嵌套层次、ViewStub延迟初始化
   >
   > 电量优化：
   >
   > - 使用工具Battery Historian分析应用耗电量
   > - CPU占用率高时耗电量也会升高，因此从观察CPU占用率开始优化
   > - 网络传输会导致耗电升高，因此也需要优化网络传输
   > - GPS定位会导致耗电升高，在用户界面不可见时要取消定位监听
   > - 谨慎使用WakeLock，它会阻碍系统休眠，耗电严重
   > - 使用传感器时，选择合适的采样率，采样率越高耗电越严重
   > - 对于可延迟的、场景触发的任务，使用JobScheduler优化，不要在子线程使用无限循环
   >
   > 业务优化：
   >
   > - 串行业务并行化
   > - 思考业务方案，简化流程

2. 事件分发机制 

   > 触摸事件从Activity#dispatchTouchEvent(MotionEvent ev)方法开始。MotionEvent事件从Activity到PhoneWindow再到DecorView，这就交给了ViewGroup#dispatchTouchEvent处理。
   >
   > 1. ViewGroup#dispatchTouchEvent采用责任链模式，在触摸事件MotionEvent.ACTION_DOWN时，将此事件交给child#dispatchTouchEvent处理，如果child的dispatchTouchEvent事件返回true代表它要处理此次事件，单向链表target记录此child的引用，在其他MotionEvent.ACTION_MOVE等事件到来时，直接交给child#dispatchTouchEvent处理。
   >2. 再看child view对触摸事件的处理，如果child是一个ViewGroup，那么它会将此事件继续交给下层child处理，一直到最下层View的dispatchTouchEvent，它返回结果首先由onTouch方法布尔返回值决定，其次由onTouchEvent方法布尔返回值决定，如果这两个方法都返回false，那View#dispatchTouchEvent方法就返回false，事件也就交还给parent ViewGroup来处理了。
   > 3. 回到ViewGroup#dispatchTouchEvent方法，child#dispatchTouchEvent返回了false，接着将触摸事件交个此ViewGroup自己处理，调用super.dispatchTouchEvent，也就是会走此ViewGroup的onTouch、onTouchEvent方法，这两个方法的返回值决定了ViewGroup自己处理触摸事件，还是将触摸事件交给上层ViewGroup或Activity#dispatchTouchEvent方法的处理结果。
   > 4. 以上是ViewGroup责任链模式处理触摸事件的大致过程，将触摸事件从Activity到View层层向下传递，到最下层View如果它不愿意接收处理事件，事件又将层层向上回传，直到返回给Activity处理。
   > 5. ViewGroup#dispatchTouchEvent交给child处理之前，会调用onInterceptTouchEvent方法判断是否拦截此事件，ViewGroup默认返回false不拦截，自定义ViewGroup可以重写此方法拦截事件，拦截之后事件不会再分发给ViewGroup的子View了。另外ViewGroup在调用onInterceptTouchEvent方法之前，会判断mGroupFlags是否包含FLAG_DISALLOW_INTERCEPT标识，如果包含那也是会拦截事件分发。FLAG_DISALLOW_INTERCEPT标识由requestDisallowInterceptTouchEvent(boolean disallowIntercept)控制，一般在子View中可以调用。总结有三种方法拦截事件的传递：
   >   1. Activity/ViewGroup#dispatchTouchEvent
   >    2. ViewGroup#onInterceptTouchEvent
   >   3. ViewParent#requestDisallowInterceptTouchEvent。

3. Binder机制，Stub类中asInterface函数作用，BnBinder和BpBinder区别。

4. 什么时候可以获得View控件的大小

   > 在Activity#onResume方法中获取不到View的宽高，可尝试一下方法获取：
   >
   > 1. View.post()：新建任务加入主线程mH的MessageQueue，当mH处理完handleResumeActivity后会处理，此时已可以获得宽高
   > 2. Activity#onWindowFocusChanged方法中获取，这个方法回调也是在绘制完成之后
   > 3. viewTreeObserver注册监听获取，注册的回调会在onLayout完成后调用
   >
   > 建议使用View.post()方式获取，其余两种方法会被调用多次。

5. 什么是过渡绘制，如何防止过渡绘制

   > 过度绘制：指系统在渲染单个帧的过程中多次在屏幕上绘制某一个像素，如界面被遮盖的元素在渲染时被绘制就发生了过度绘制。
   >
   > “开发者选项-调试GPU过度绘制-显示过度绘制区域”，Android将为界面元素着色：
   >
   > - 真彩色：没有过度绘制
   > - 蓝色：过度绘制1次
   > - 绿色：过度绘制2次
   > - 粉色：过度绘制3次
   > - 红色：过度绘制4次或更多
   >
   > 解决过度渲染问题：
   >
   > - 移除布局中不需要的背景，可以快速提高渲染性能；
   > - 减少使用透明度：透明度是一种混合效果，系统会绘制像素多次；
   > - 使视图层次结构扁平化、采用开销较低的布局：采用ConstraintLayout、merge/include等，未设置权重的LinearLayout也优于RelativeLayout；
   >
   > Double Taxation：指复杂布局需要measure-layout多次才能够确定最终的元素位置。例如使用RelativeLayout、水平方向LinearLayout、有权重LinearLayout时，framework会执行一下操作：
   >
   > - 执行一次measure-layout遍历以确定子元素的位置和大小
   > - 结合数据和对象的权重确定关联对象的位置
   > - 执行第二次布局遍历以确定最终位置
   > - 进入渲染过程的下一阶段
   >
   > 视图结构的层次越多，潜在的性能消耗就越大。

6. Acticity的生命周期以及四种启动模式

   > 生命周期：onCreate - onStart - onResume - onPause - onStop - onDestroy
   >
   > 旋转屏幕：onPause - onStop - onSaveInstanceState  - onDestroy - onCreate - onStart - onRestoreInstanceState - onResume
   >
   > 屏幕熄灭亮起：onPause - onStop - onSaveInstanceState - onRestart - onStart - onResume
   >
   > singleTop、singleTask启动模式复用Activity：onPause - onNewIntent - onResume
   >
   > Activity A 启动 Activity B：A onPause - B onCreate-onStart-onResume - A onStop
   >
   > 
   >
   > 平时我们使用第三方库，往往需要在activity生命周期中调用绑定/解绑、初始化/释放等操作，虽然写法简单但是当接入的第三方库一多会使得代码难以维护。
   >
   > androidx.lifecycle包中Activity/Fragment都实现了LifecycleOwner接口，用户可以通过获取Lifecycle对象来设置生命周期的观察者LifecycleObserver，在生命周期回调发生时会触发相应的方法。
   >
   > Lifecycle使用两个枚举来跟踪生命周期状态：Event、State。
   >
   > State包括INITIALIZED DESTROYED CREATED STARTED RESUMED
   >
   > Event包括ON_CREATE ON_START ON_RESUME ON_PAUSE ON_STOP ON_DESTROY
   >
   > State是表示一种状态，Event是State状态之间切换时触发的事件。自定义LifecycleObserver类可以通过在方法上使用@OnLifecycleEvent注解，当制定的Event发生时就会回调到注解的方法，方法中也可以通过lifecycle.currentState获取当前所处的State。

7. Service的两种启动模式及各自对应的生命周期

   > Service是一种即使用户未与应用交互也可以在后台运行的组件，运行在主线程。如果后台任务必须在主线程之外执行，并且存在用户与应用交互时使用，那么应该创建新线程执行任务，在onStart中启动线程，在onStop中停止线程。否则才使用Service，如果任务是耗时的，那么应在Service中创建新线程。
   >
   > 生命周期：
   >
   > - onStartCommand 调用startService启动服务后会调用此方法，启动的服务需要调用stopService/stopSelf来停止服务，否则会无限期运行
   > - onBind 调用bindService绑定服务后会调用此方法，必须通过返回IBinder提供一个接口给客户端来与服务端通信。
   > - onCreate 启动/绑定服务都会调用的方法，发生在onStartCommand/onBind之前
   > - onDestroy 服务注销时的回调方法，应该重写此方法，在其中清理资源

8. volley的源代码

   > volley API中有两个很重要的类，一个是RequestQueue、一个是Request，使用流程为：
   >
   > 1. 创建RequestQueue，如Volley.newRequestQueue()；
   > 2. 创建Request，包含请求方式、url、成功回调、失败回调、优先级mSequence值；
   > 3. 调用RequestQueue#add(Request)执行请求；
   >
   > Volley源码分析：
   >
   > 1. RequestQueue：构造函数会传入缓存配置、网络配置（httpurlconnection/httpclient），之后调用start方法创建一个CacheDispatcher和默认的4个NetworkDispatcher线程，并调用其start方法。一般通过单例模式使用RequestQueue，避免重复创建多个分发线程。RequestQueue中还包含两个优先级阻塞队列PriorityBlockingQueue，一个存储缓存Request的mCacheQueue、一个存储网络Request的mNetworkQueue。
   >
   > 2. 当调用RequestQueue#add方法加入Request时，如果Request不可缓存则将Request加入mNetworkQueue，否则加入mCacheQueue。CacheDispatcher中的run方法有一个无限while循环，不断从mCacheQueue中取Request处理：如果根据此Request的key获得的缓存实体Entry的Expires没有过期那么将缓存结果返回，否则将Request加入mNetworkQueue。NetworkDispatcher中的run方法同样也有一个无限while循环不断从mNetworkQueue中取Request，取出后交给HttpUrlConnection/HttpClient访问网络获取响应结果，成功后还会将响应预元数据存放在Cache中。
   >
   > 3. 补充：PriorityBlockingQueue它是一个基于堆的无界、并发安全的优先级队列。其中的元素不能为null，且必须实现Comparable接口，compareTo方法返回负数表示优先级高、返回正数表示优先级低，插入元素时会将优先级高的放在队列前面，优先级低的放在队列后面，取出元素时从优先级高的元素开始，当队列中无元素时调用取出操作take()会阻塞线程。
   >
   > volley与okhttp比较：
   >
   > volley是基于httpurlconnection/httpclient，并封装了图片加载框架，其使用的缓存是基于Expires过期时间来判断，是一个轻量级的网络框架（不适合下载大量内容，官网推荐的是DownloadManager）；而okhttp是基于socket和okio，拥有自动维护的socket连接池（减少握手次数）、队列线程池、基于Header的缓存策略，是更高效的网络请求框架。

9. fragment的生命周期

   > onAttach：与Activity关联时调用
   >
   > onCreate：系统创建Fragment时调用
   >
   > onCreateView：创建与Fragment关联的View
   >
   > onActivityCreated：Activity#onCreate方法已返回
   >
   > onStart - onResume - onPause - onStop
   >
   > onDestroyView：移除Fragment关联的View时调用
   >
   > onDestroy：系统销毁Fragment时调用
   >
   > onDetach：取消与Activity关联时调用

10. 手机适配一些方案

    > 屏幕适配问题：
    >
    > Android提供dp单位替换px来做屏幕适配，换算单位是px = density * dp。density的计算方式是像素密度dpi / 160，像素密度是屏幕尺寸上的像素数 / 屏幕尺寸。使用dp来适配，在不同机型上难以达到设计图所示的效果，比如设计图的宽是360dp，我们一台1920*1080、5寸的手机计算density = 440dpi / 160，而屏幕宽的dp值则是1080px / (440dpi / 160) = 392.7dp，但我们如果按照设计图360dp的宽度来设置UI的话，真实效果显然是会和设计图不同的，各种其他机型的真机效果也是千差万别了。为了解决这个问题，今日头条屏幕适配方案是：仍旧使用dp来适配、通过修改系统density值要让屏幕宽或高其中一个计算的dp值与设计图相等。比如设计图宽度为360dp，不同机型设置它自己的density，使得不同屏幕宽度px值 / density = 360dp，这样就能使得不同机型UI完整设配设计图了。修改density的方式是：因为我们dp和px转化都是通过DisplayMetrics相关的api来的，因此只需修改Activity和Application中Resource中的DisplayMetrics的density、scaledDensity、densityDpi值即可，而修改后的density = displayMetrics.widthPixel / 设计图宽度360dp；修改后的scaledDensity根据原有scaledDensity与density比值决定，使得修改后保持同一比值；修改后的densityDpi则等于density/160。最后在Activity#onCreate方法中调用修改的方法即可。
    >
    > 
    >
    > 状态栏适配问题：
    >
    > 状态栏透明，内容布局延伸到系统状态栏达到沉浸式状态栏效果，在Android各版本设置的方式不同。
    >
    > - Android 5.0及以后版本：设置getWindow().getDecorView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN)使内容延伸到状态栏，再通过setStatusBarColor把系统状态栏设置成透明即可；
    > - Android 4.4 - Android 5.0：通过getWindow().addFlag(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)可以让状态栏透明并且我们内容延伸到系统状态栏；
    > - Android 4.4以前不支持透明状态栏
    >
    > 我们有时并不需要沉浸式状态栏，只需状态栏颜色与我们toolbar颜色相同即可。
    >
    > - Android 5.0及以后版本：直接设置setStatusBarColor()或修改colorPrimaryDark值
    > - Android 4.4 - Android 5.0版本：通过getWindow().addFlag(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)让状态栏透明后，我们自己的内容布局向上设置一个等于系统状态栏高度的padding即可
    >
    > 另外在Android6.0开始提供了修改状态栏字体颜色的方法，避免因自定义状态栏颜色使得字体不可见的问题。字体颜色默认白色，可修改为黑色：getWindow().getDecorView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR)

11. Android资源管理

    > Android Studio的资源管理窗口：View - Tool Window - Resource Manager，可以在其中将可绘制对象导入资源（自动解析密度），也可以将资源直接拖到xml布局中显示。
    >
    > Android的资源管理框架：Resources和AssetManager，分别管理res和assets目录下的资源。res包下除了raw包以外的资源都会通过aapt编译成二进制格式的xml文件，注册到R类中并生成resources.arsc索引文件，其他assets和res/raw下的资源不做处理，使用时需要读取原文件。

12. 类加载器机制、加载流程、 双亲委托机制，类的五个加载过程。

    > 类加载机制：虚拟机把描述类的数据从Class文件中加载到内存，并对数据进行校验、转换解析和初始化，最后形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。Java语言中，类型的加载、连接和初始化过程都是程序运行期间完成的，这样虽然会令类加载时稍微增加性能开销，但是也为Java程序提供了很高的灵活性，Java可以动态扩展的语言特性就是依赖运行时动态加载和动态连接这个特点实现的。
    >
    > 
    >
    > 类加载时机：一个类从被加载到虚拟机内存开始，到卸载出内存，其生命周期是：加载、验证、准备、解析、初始化、使用和卸载七个阶段。什么时候需要类加载并没有强制约束，但是对类初始化却有严格规定，而初始化之前必须对类进行加载、验证、准备、解析过程。初始化场景有且只有在5个主动引用情况下发生。
    >
    > - 遇到new、getstatic、putstatic或invokestatic这4条指令且对应的类没有初始化时
    > - 对类进行反射调用且没有初始化时
    > - 初始化一个类，需要先触发其父类的初始化且父类没有初始化时
    > - 虚拟机启动时，用户指定的main方法所在的类的初始化
    > - MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，且这个方法句柄对应的类没有进行过初始化时
    >
    > 除此之外都不会触发初始化，其引用被称为被动引用。如
    >
    > - 通过子类引用父类的静态字段，只会初始化父类，不会初始化子类
    > - 创建此类的数组对象，不会触发此类的初始化
    > - 常量在编译期间会存入调用类的常量池，因此也不会初始化常量定义的类
    >
    > 
    >
    > 类加载过程：
    >
    > - 加载
    >   - 通过一个类的全限定名来获取此类的二进制字节流
    >   - 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
    >   - 在内存中生成一个代表这个类的Class对象，作为方法区访问这个类的各种数据的访问入口
    > - 验证：目的是确保Class文件的字节流中的信息符合当前虚拟机的要求
    >   - 文件格式验证（验证字节流是否符合Class文件格式规范）：魔数开头、主次版本号、检查常量池tag标记是否支持、检查各种索引是否指向不存在的常量或不符合类型的常量、CONSTANT_Utf8_info是否有不符合UTF-8编码的数据等
    >   - 元数据验证（对字节码描述的信息进行语义分析）：这个类是否有父类、是否继承不允许被继承的类（final修饰）、如果不是抽象类是否实现了父类所有抽象方法、类中的方法、字段是否与父类产生矛盾等
    >   - 字节码验证（对类的方法体进行校验分析，保证方法运行时不会危害虚拟机安全）
    >   - 符号引用验证（发生在准备阶段，将符号引用转换为直接引用时，对常量池中各种符号引用的信息进行匹配校验）：如符号引用描述的全限定名能否找到对应的类、指定类中是否存在对应的字段和方法、指定的类的字段和方法是否可以被访问
    > - 准备：正式为类变量分配内存并设置类变量初始值，需要注意的是只会为类变量（static修饰）分配内存，且初始化的值一般为0
    > - 解析：将常量池中的符号引用替换为直接引用
    >   - 符号引用：是用一组符号来描述所引用的目标，可以是任何字面量，只要能定位到目标即可
    >   - 直接引用：是可以指向目标的指针、相对偏移量或一个能间接定位到目标的句柄。
    >   - 解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这7种符号引用进行
    >   - 类或接口的解析过程是：如果不是数组，那么将它的全限定名传递给类加载器加载，期间可能会触发其他相关类的加载，加载完成之后，就有这个类的直接引用了。
    > - 初始化：初始化阶段是执行类构造器<clinit>()方法的过程
    >   - <clinit>()方法是由编译自动收集所有类变量的赋值动作和static静态代码块中语句合并生成
    >   - <clinit>()不需要显示调用父类的<clinit>()方法，虚拟机会保证父类的方法先执行
    >   - 虚拟机会保证<clinit>()在多线程环境下是加锁、同步的。
    >
    > 
    >
    > 双亲委托机制：
    >
    > Java系统提供三种类加载器：启动类加载器、扩展类加载器、应用程序类加载器，双亲委托机制要求除了顶层启动类加载器外，所有其他类加载器都应该有自己的父类加载器。如果一个类收到类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的类加载请求都应该传到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个类加载请求时，子加载器才会尝试自己去加载。

13. View的绘制流程

14. EventBus源码

15. 内存泄露和内存溢出

16. Serializable和Parcelable的区别

17. ListView或者RecyclerView优化

    > 1.不要在onBindViewHolder中设置点击事件或执行耗时操作
    >
    > 2.RV嵌套RV的优化：使用LinearLayoutManager.setInitialPrefetchItemCount()设定初次显示时可见item的个数
    >
    > 3.一个ViewPager中包含多个同列表的Fragment，可以通过RecyclerView#setRecyclerViewPool这些RV共用RecyclerViewPool；
    >
    > 4.使用DiffUtil精确刷新
    >
    > 5.当RV使用onItemRangeChanged/Inserted/Removed/Moved等方法时，会触发requestLayout重新计算item大小，如果item宽高固定，则可以使用setHasFixedSize(true)来避免RV重新计算
    >
    > 6.优化Item布局，减少嵌套
    >
    > 7.数据非常少时，可以使用ListView，减少内存消耗

18. 什么是ANR，Activity、BroadcastReceiver、Service对ANR时间限制分别是多少，怎么处理ANR，除了系统生成trace.txt文件，怎么在程序中检测ANR。写出伪代码

    >如果应用的主线程处于阻塞状态，无法响应用户输入事件或绘制操作，一段时间后就会触发“应用无响应”，也就是ANR错误。
    >
    >时间限制分别是：5s,10s,20s
    >
    >ANR一般由以下几种情况产生：
    >
    >1.在应用主线程做耗时操作，比如I/O操作、长时间的计算；
    >
    >2.主线程调用另一个进程的binder服务，后者长时间没有返回；
    >
    >3.主线程处于阻塞状态，正在等待另一个线程的锁的释放，或者形成了死锁。
    >
    >处理ANR：在anr发生时通过adb pull /data/anr/<filename>拉取trace.txt文件分析anr原因；程序可以正常运行的情况下，也需要通过Android Studio的Profiler工具分析主线程的运行情况，寻找耗时的方法，将耗时操作放到异步线程中处理。
    >
    >程序检测anr：Looper在loop()方法中循环获取到一个Message后，会调用mLogging打印日志，执行完Handler#dispatchMessage()方法后，会再次调用mLogging打印日志。mLogging是Looper中的Printer类型的成员，我们可以通过Looper.setMessageLogging(Printer printer)赋值自定义的Printer，在自定义Printer中计算两次println()方法的时间差。BlockCanary原理就是通过自定义Printer，监测到耗时超过默认值3000ms时，记录并发送通知提醒。
    >

19. Android的类加载器

    > PathClassLoader只能加载系统安装过的apk
    >
    > DexClassLoader可以加载jar/apk/dex，可以从SD卡加载未安装的apk
    >
    > 两者的区别在于PathClassLoader传入optimizedDirectory为null，最终导致只能加载安装过的apk（具体源码要追踪到native文件）
    >
    > 两者的findClass方法都在父类BaseDexClassLoader中，findClass()调用DexPathList#findClass，其中通过遍历dexElements数组得到DexFile，再通过DexFile#loadClassBinaryName加载Class

20. 插件化原理

    > 通过Android的类加载机制，加载jar、apk或dex插件包中的class文件，并通过反射调用其中的方法。
    >
    > 插桩方式实现插件化：
    >
    > ​	1.反射调用AssertManager的addAssetPath加载apk包，获得Resource实例；
    >
    > ​	2.建立自定义生命周期接口，宿主、插件都要实现自定义生命周期方法，宿主创建ProxyActivity、ProxyReceiver、ProxyService并在清单文件中注册占位，当需要打开插件Activity时，打开ProxyActivity并传入插件Activity的路径，反射获得插件Activity实例，并在各生命周期方法中调用插件Activity的自定义生命周期方法；重写getResources和getClassLoader替换为插件包的Resource和ClassLoader。
    >
    > 缺陷：宿主和插件apk依赖性强，插件crash同时会导致宿主crash

21. Handler异步消息机制

    > 四个重要的类：Handler、Message、Looper、MessageQueue
    >
    > Handler发送消息Message，会将自身实例传入Message中保存，Message加入到MessageQueue，MessageQueue是一个单链表形式的结构，根据Handler#postDelay的时间插入到MessageQueue正确的位置。Looper是一个线程唯一实例的类，使用Handler必须传入Looper实例，Looper的prepare方法绑定线程，loop方法开启循环，不断从MessageQueue中取出Message实例，再通过Message中的保存的Handler对象，调用Handler的dispatchMessage方法，这个方法优先执行存在Runnable的回调，否则执行Handler的handleMessage方法。无论在哪个线程通过Handler发送消息，处理消息的线程都是此Handler中的Looper初始化绑定的线程，正是在这个绑定线程调用的loop方法，不断轮询消息处理。
    >
    > Android的消息机制即是在ActivityThread中存在一个主线程的Handler对象mH，Android的事件驱动指的就是此mH发送、处理事件，在ActivityThread的main方法中会初始化主线程的Looper，调用prepare和loop方法开启主线程的loop循环，处理事件。在loop循环中，通过MessageQueue不断的获取Message，之所以没有导致主线程卡死，是因为Android本身就是事件驱动的，主线程正是在loop循环中等待下一个事件到来。Activity、Service的生命周期回调就是通过mH实例发送相关Message调用的。另外此循环之所以不是特别消耗cpu资源，是因为MessageQueue保持了一个native层的MessageQueue引用，并且存储、取出Message的方法也都是依赖native层实现的，在native层依赖Linux的管道epoll机制，阻塞的时候会释放cpu资源。
    >
    > 补充：
    >
    > - MessageQueue中的一个IdleHandler集合，通过给主线程Looper的MessageQueue添加IdleHandler回调，可以实现在Activity页面绘制完成、所有Message处理完毕时，执行其他业务。
    > - Looper的loop循环，在MessageQueue取出一个Message后到Handler处理完这个事件之后，mPrinter对象都有打印日志，可以通过Looper设置自定义的Printer对象，实现对Handler处理消息的时间监听。
    > - Handler持有Activity对象造成内存泄漏的原因：如果Handler直接持有Activity的引用，那当Activity关闭之后，如果MessageQueue中仍存在一个因为需要延时而存在的Message，又由于Message持有Handler引用，因此也就间接持有了Activity的引用，这样就导致Activity内存泄漏了。解决方法是让Handler持有Activity的虚引用。
    > - Handler的同步屏障机制：MessageQueue#postSyncBarrier会在MessageQueue中根据时间位置插入一个没有Handler引用的Message，当MessageQueue中取到没有Handler引用的Message时，会进入一个do-while循环，通过msg.next遍历寻找到一个异步（isAsynchronous）消息后跳出循环去执行此异步消息。这就像是给MessageQueue中的同步消息插入了一个屏障，遇到这个屏障时使得异步消息能够优先执行，View的绘制任务就是使用同步屏障机制的异步消息。

22. Handler机制，HandlerThread实现

    > HandlerThread内部封装了一个Handler实例，方便在子线程中使用Handler消息机制。当线程的run方法执行时，会创建Looper，调用loop方法开启循环。

23. LRUCache算法

    > LRUCache内部封装了一个LinkedHashMap。LinkedHashMap是在HashMap的基础上，新增了一个双向循环链表，将LinkedHashMap中所有节点连接起来。LinkedHashMap有两种排列方式，一种通过访问时间排序，一种通过插入时间排序。LRUCache内部的LinkedHashMap使用访问时间排序，最近访问的排在列表最前面，当LRUCache#put一个元素后，先判断Map中是否已存在相同key的元素，如果不存在则放入Map中，之后会计算LRUCache的size是否大于maxSize，如果大于，则通过遍历LinkedHashMap，取出最后一个元素删除，保证最先被访问的元素被移除。

24. 统计启动时长和如何优化冷启动时间

    > 本地启动时间统计：通过**adb shell am start -W 包名/全路径类名**启动Activity，将打印Activity启动时间；或者查看日志，tag是ActivityManager的Displayed时间；另外还可以调用reportFullyDrawn()方法来自定义Activity什么情况下才算完全展示，系统也会打印tag是ActivityManager的完全展示时间日志。
    >
    > 线上启动时间统计：通过日志埋点，在Application#attachBaseContext方法中记录起始点，在Activity#onWindowFocusChanged方法中记录结束点，上报启动时间。
    >
    > 冷启动优化：
    >
    > 1.通过设置主题背景，去除黑白屏，优化用户体验
    >
    > 2.MultiDex启动优化，Android 5.0以下使用Dalvik虚拟机，MultiDex加载分包时耗时严重，可以采用预加载方案，另开进程加载分包；Android5.0以上使用Art虚拟机，apk在安装的时候已经执行过预编译成.oat文件，以备Android设备执行，MultiDex是默认开启的，无需手动添加
    >
    > 3.Application#onCreate()方法中，需要进行初始化的组件和第三方库，具体分析是否可以放在子线程初始化、是否可以延迟到启动页初始化或者是否可以到用的时候才初始化
    >
    > 4.启动页Activity初始化：对于布局，除了启动图之外，一些闪屏广告、视频、第一次安装介绍图等不是必须展示的控件，通过ViewStub优化；对于sp、数据库的使用，放在异步线程，避免主线程阻塞；对于从Application中延迟到启动页的初始化操作，可以放在Activity#onWindowFocusChanged方法中，也可以放在主线程的IdleHandler中，只要不影响界面显示都可以。

25. activity栈的应用场景

    > 首先要了解任务和任务栈，任务是用户操作Activity的集合，这些Activity按照打开顺序排列在一个任务栈中，新加入的Activity会放在栈顶并获得焦点，上一个Activity仍然位于栈中，但处于停止状态。按返回键退出Activity时，该Activity也会从任务栈中退出，之后系统显示任务栈新的栈顶Activity。当用户按Home键回到桌面，任务会转到后台，此时用户只能通过点击桌面图标来恢复该任务。
    >
    > Android也可以通过其他方式来管理任务和任务栈，如启动模式、taskAffinity改变亲和性、allowTaskReparenting切换任务栈、clearTaskOnLaunch启动时清空任务栈中根Activity以外的Activity、alwaysRetainTaskState在任务回到后台一段时间后仍保存所有任务栈中的Activity（默认系统会使用clearTaskOnLaunch清除Activity）、finishOnTaskLaunch任务转到后台此Activity就被清除。
    >
    > 关于亲和性指同一个应用中所有Activity都具有亲和性，都会倾向于存在同一个任务栈中。但是taskAffinity属性可以指定同一应用Activity具有不同的亲和性，或不同应用Activity具有相同亲和性，其值是一个字符串，默认是manifest中的包名。亲和性不能应用于singleTask和singleInstance启动模式的Activity，因为这两类Activity必须位于任务栈的根部。FLAG_ACTIVITY_NEW_TASK标记需要搭配taskAffinity指定除manifest包名外字符串使用才有效，否则通过FLAG_ACTIVITY_NEW_TASK打开的Activity仍位于原任务栈中。taskAffinity也常与allowTaskReparenting配合使用，如果某个前台Activity的allowTaskReparenting=true，那么当用户按Home键回到桌面再点击某个桌面图标打开其他应用时，如果这个被打开的应用的亲和性与之前的Activity相同，那么该Activity会被转移到新打开的应用中。

26. 封装view的时候怎么知道view的大小

    > 自定义View绘制前会经历一系列方法：构造函数、onFinishInflate、onMeasure、onSizeChanged、onLayout、onWindowFocusChanged等。子View的引用在构造函数中无法获取，需要等到onFinishInflate调用时才能拿到。onMeasure中可以获取到measuredWidth和measuredHeight值，无法获取width、height值，但是onMeasure可能会多次调用，获取到的值有可能会发生改变。在onSizeChanged方法是控件宽高发生变化时的回调，和onMeasure一样也可以获得measuredWidth、measuredHeight。onLayout是确定摆放时的回调，此时才能获取到width和height值。onWindowFocusChanged是View的焦点发生改变时的回调，此时已经绘制完成，也可以获取到宽高。
    >
    > 另外，可以通过addOnGlobalLayoutListener监听onLayout回调发生后，获取到宽高，需要注意remove监听。

27. 进程间通信的方式

    > Activity、Service、广播、Content Provider四大组件之间的通信都可以跨进程，通信通过Bundle传递基本数据类型或能够序列化的数据。
    >
    > 通过AIDL方式：服务端定义aidl接口、定义Serivce在onBind方法中提供实现了aidl接口IBinder对象；客户端程序放入拷贝的aidl文件到同名目录下，通过bindService绑定服务端服务，在onServiceConnected方法中调用asInterface将IBinder对象转化为aidl接口实例，从而能够调用aidl接口中定义的方法。
    >
    > 其他通过文件共享、SharedPreference、Messager、Socket方式，用的不多。

28. 计算一个view的嵌套层级

    > View的getParent方法将返回父布局的View实例，Activity最外层View是DecorView，DecorView的getParent对象返回的是ViewRootImpl实例，而ViewRootImpl的getParent返回的是null。因此可以通过嵌套调用View的getParent方法计算View的层级。
    >
    > 从源码分析一下为什么DecorView的getParent方法返回的是ViewRootImpl对象：在ActivityThread的handleResumeActivity方法中会调用ViewManager的addView方法，参数是DecorView和WindowManager的Layout属性，addView具体实现在WindowManagerGlobal中，创建ViewRootImpl实例后，调用setView方法，参数包含此DecorView，并且会调用DecorView的assignParent方法，将ViewRootImpl赋值为DecorView的mParent。

29. 动态权限适配方案

    > EasyPermissions框架
    >
    > 1.onRequestPermissionsResult回调需要转发给EasyPermissions处理
    >
    > 2.hasPermissions判断是否已申请权限，requestPermissions申请权限
    >
    > 3.使用@AfterPermissionGranted(RequestCode)注解，EasyPermissions处理权限申请回调后会调用注解的方法。
    >
    > 4.通过PermissionRequest.Builder自定义权限申请对话框
    >
    > 5.也可以实现onPermissiosGranted和onPermissionsDenied回调
    >
    > 6.通过RationaleCallbacks动态申请权限前先弹出对话框，提示用户为什么需要这些权限

30. 网络请求缓存处理，okhttp如何处理网络缓存的

    > HTTP缓存是通过请求头与响应头中配置的。
    >
    > 缓存分两种，强制缓存和对比缓存。
    >
    > 强制缓存是服务端响应头中返回Expires到期时间，或者Cache-Control代替Expires。Cache-Control:max-age=60 表示60秒后缓存过期，下一次请求时，如果请求时间小于过期时间，将直接使用缓存。强制缓存和对比缓存如果同时出现，那么强制缓存优先级更高。
    >
    > 对比缓存是响应头Last-Modified、请求头If-Modified-Since或响应头ETag、请求头If-None-Match。Last-Modified是服务端返回的资源最后修改时间，下一次请求时将此时间放入If-Modified-Since请求头中给服务端对比，如果资源一直没有改动过，则响应码事304告诉客户端使用缓存，否则返回200响应所有内容。ETag是服务端响应头中告诉客户端的资源标识符，下次请求时If-None-Match中带上这个标识符，服务端判断资源是否过期。ETag/If-None-Match优先级高于Last-Modified/If-Modified-Since。
    >
    > okhttp通过CacheInterceptor处理缓存，缓存策略由CacheStrategy决定，缓存内容由Cache类管理。
    >
    > Cache内部有一个DiskLruCache，通过构造方法传入缓存路径之后，就可以通过定义的增删改查方法操作缓存内容。在put方法中，会过滤掉非GET请求的响应结果，因此只能缓存GET请求的结果。CacheStrategy通过工厂模式创建，Factory构造需要传入请求Request对象和从Cache取出的缓存结果Response对象，获取实例时在get方法中根据cacheResponse中的请求头中的过期时间或ETag标识会返回有条件的缓存策略。
    >
    > CacheInterceptor中根据缓存策略中取出的networkRequest和cacheResponse采取不同的处理方式。如果networkRequest为null、cacheResponse为null那么返回504；如果networkRequest为null、cacheResponse不为null那么返回缓存结果；如果networkRequest不为null，那么将networkRequest交给下一个拦截器ConnectInterceptor与服务器进行连接，得到返回结果networkResponse，如果cacheRespnose不为null，会将networkResponse和cacheResponse进行比较，判断缓存是否过期，未过期的话返回缓存结果，否则返回networkResponse并更新缓存内容。

31. Android系统为什么会设计ContentProvider，进程共享和线程安全问题

    > ContentProvider存在的意义在于给其他应用提供数据访问的接口，所以也是跨进程的。
    >
    > 多个进程同时访问时，ContentProvider提供方只存在一个实例，通过多线程并发操作。操作不具备原子性，考虑到线程安全问题可以在更删改查操作方法中加锁。

32. 动态代理模式

    > 动态代理涉及两个类，一个是Proxy，一个是InvocationHandler。
    >
    > Proxy的newProxyInstance方法，用于构造实现指定接口的代理类实例，所有方法的调用都会给指定的InvocationHandler的invoke方法处理。InvocationHandler需要持有被代理类的实例，在invoke方法中根据method手动调用被代理类对应的方法，可以在调用前后做其他处理逻辑。
    >
    > 动态代理与静态代理都需要持有被代理的实例，区别在于，静态代理类需要实现被代理类的公共接口，实现的方法众多，不便于维护管理；另外动态代理是InvocationHandler持有被代理类实例，而静态代理则是代理类持有被代理类实例。
    >
    > 关于动态代理类，根据Java虚拟机类加载机制，加载类是通过一个类的全限定名来获取此类的二进制字节流，这个加载过程是很灵活的，可以从文件中获取、从网络中获取，也可以运行时计算生成，动态代理生成的类正式运行时计算生成的。

33. App 是如何沙箱化，为什么要这么做

    > Android平台是利用基于用户的Linux保护机制来识别和隔离应用资源。Android会为每一个应用分配一个独一无二的用户ID（UID），并在自己的进程中运行，都拥有自己独立的虚拟机实例。在没有权限的情况下，UID不同的应用之间不能彼此交互，也不能访问系统的资源。
    >
    > Android依靠许多保护机制来强制执行应用沙盒：
    >
    > Android5.0使用安全增强型Linux（SELinux）对所有进程执行强制访问控制（MAC），应用于系统分隔开，任何未经明确允许的行为都将被拒绝；Android6.0SELinux沙盒进一步扩展，可以跨各个物理用户边界来隔离应用，降低了可以从设备或系统之外的访问，同时还为应用数据设置了更安全的默认设置；Android8.0通过一个过滤器限制应用可以使用的应用调用；Android9所有非特权应用必须在不同的SELinux沙盒进行；Android10应用对文件系统了解受限，无法直接访问/sdcard/DCIM之类的路径。

34. Activity的启动模式

    > standard 默认，按照栈的特点进栈出栈。singleTop 避免当前显示的栈顶Activity被重复创建，会走onNewIntent回调传递intent，适用于通过通知栏推送点击跳转的Activity。
    >
    > singleTask 和 singleInstance的启动的Activity都是位于任务栈的根部，区别在于singleTask其上仍然可以加入standard和singleTop的Activity，而singleInstance不允许加入任何Activity。singleTask模式在已存在Activity的情况下，会通过onNewIntent传递intent，并清空栈顶，常用于App的主页。singleTask和singleInstance模式建议在Activity拥有action是android.intent.action.MAIN、category是android.intent.category.LAUNCHER的intent-filter的情况下使用，也就是说此Activity最好拥有一个在桌面的入口，否则当用户按下Home键任务栈退回后台后，用户就找不到这个Activity了。

35. Activity是如何缓存的

    > 在用户按Home键回桌面、或选择打开了其他应用、或关闭屏幕、屏幕方向转化、启动其他Activity时都将调用Activity的onSaveInstanceState方法，可以通过Bundle保存临时数据和状态，当Activity被清理重启时，会回调Activity的onRestoreInstanceState方法，或通过onCreate方法，这两个方法的Bundle参数都可以获取到之前保存的临时数据和状态，区别在于onRestoreInstanceState方法在onStart之后调用（onSaveInstanceState方法也是在相应的onStop方法之前调用的）。View也实现了onSaveInstanceState方法，只要给View设置id，在Activity重启后也能自动回复之前的状态。onSaveInstanceState方法的调用存在很大不确定性，因此存储持久化的数据最好放在onStop中，Activity打开下一个Activity的生命周期流程是先执行前一个Activity的onPause方法，要等到下一个Activity的onResume方法执行完成之后才会执行前Activity的onStop方法，因此如果在主线程中存储持久化数据的操作放在onPause方法中是会阻塞下一个Activity显示的。

36. 进程保活

    > 对于进程保活需要了解一下应用进程生命周期。Android应用运行在Linux进程中，它的生命周期并不是由自身直接控制的，而是由系统各方面控制，如应用对用户的重要程度、系统中内存的使用情况等。应用的Activity、服务、广播组件的使用方式，对进程的生命周期有很重要的影响。比如在广播接收者中开启一个线程后执行完onReceive方法的返回，那么系统将认为此广播已不再处于活动状态，对于没有其他应用组件活跃的进程，系统可能会随时终止该进程，通常通过调用JobService来替换开线程，系统就知道这个进程中还有活跃的任务。
    >
    > Android会根据运行的组件状态来区分进程的重要程度：
    >
    > 前台进程是目前用户正在操作的进程，包括屏幕上获得焦点的Activity、正在运行onReceive方法的广播以及正在执行生命周期方法的Service；可见进程指用户知晓可见的进程，包括用户可见但未获得焦点的Activity、调用Serivce#startForeground通知栏显示的服务。前台进程和可见进程对于用户来说是非常重要的进程，系统一般不会清理它们。
    >
    > 服务进程是用户无法直接看到的进程，如网络数据的上传下载，系统在没有足够内存保证前台进程和可见进程的情况下，会终止服务进程来释放内存空间。缓存进程是用户当前不需要的进程，如任务已经退到后台的进程，这些缓存进程保存在一个类似LruCache的列表中，当需要清理进程释放内存时，最先被使用到的进程将被第一个清理。
    >
    > 进程的重要程度不是一成不变的，长时间运行的服务进程会被降到缓存进程，缓存进程也可能因为bindService或ContentProvider访问而提升重要程度。

37. 简述Android IPC机制

    > 见27题

38. 注解和apt、annotationProcessor

    > apt指的是Java编译时注解处理器，位于javax包下，不能在Android工程中使用，需放在Java工程下。
    >
    > apt技术广泛用于butterknife、dagger2、eventbus、arouter等Android框架中，通过apt在编译期间根据注解生成特定的代码类，省去手动编写的麻烦。一般配合JavaPoet使用。
    >
    > 使用apt技术的架构一般是包含三个库，一个Java的注解库、一个Java的apt编译库、以及一个提供api的Android库，app的module通过annotationProcessor依赖apt编译库。注解库定义注解，编译库中创建一个类继承javax包下的AbstractProcessor，再重载process方法根据注解内容进行自定义的处理。
    >
    > AbstractProcessor的init方法中可以获取一些工具类，如操作文件读写的Filer实例，打印日志的Messager实例，通过JavaPoet的api，如TypeSpec构建类、FieldSpec构建成员变量、MethodSpec构建方法、CodeBlock构建方法体等，最后通过JavaFile构建文件对象再写入Filer流中保存到文件。
    >
    > 自定义的Processor需要注册，通过AutoService库的AutoService注解可以帮助我们自动注册。
    >
    > 自定义的Processor也可以断点调试，需要通过编辑configuration新建Remote配置，再通过gradlew命令开启调试。
    >
    > 举例butterknife中apt的使用：在process方法执行时会遍历出@BindView注解成员的类，通过JavaPoet生成一个新的类，并生成传入Activity和DecorView实例的方法，其代码是当在activity调用ButterKnife.bind(this)传入Activity和DecorView实例时，通过Activity反射获取@BindView注解的成员，获取注解的值即View的id，并通过DecorView的findViewById获得View的实例最后给Activity中的成员赋值。设置点击事件的注解也是同理。

39. MVP架构、MVVM架构、Clean架构等等

    > 在MVC框架的基础上，为了解决Activity/Fragment中业务代码过多、即是V层又是C层的问题，MVP框架将Activity/Fragment设置为V层，V层与负责管理业务逻辑的P层交互，P层与负责管理数据的M层交互，V层和M层解耦，P层负责通过M层获取数据又将获得的数据交给V层显示。
    >
    > MVP架构的优点是逻辑清晰，V层与M层解耦，UI和业务代码独立开发，维护也更方便；缺点是P层和V层都是一对一的关系，代码量大大增加，并且P层持有V层的引用却对V层的生命周期无感知，因此需要注意释放以免造成V层内存泄漏。
    >
    > MVVM架构是MVP架构的基础上，用VM层代替P层，V层（Activity/Fragment）仍然和M层解耦。xml文件中使用DataBinding，V层持有binding引用和ViewModel引用，并给ViewModel中通过M层获取的通过LiveData包装的数据添加观察者，在数据有更新的回调中给binding中的控件赋值，从而实现数据驱动UI。LiveData是使用类似于RxJava观察者模式的类，注册观察者时需要传入LifecycleOwner，使得LiveData具有Activity/Fragment的生命周期感知能力。
    >
    > MVVM最大优点是数据驱动UI，缺点是有一定的学习成本，使用LiveData、DataBinding等新类可能会遇到无法理解的问题。
    >
    > Clean架构不了解。

40. Kotlin协程

    > 协程是Kotlin中类似线程的概念，但是更轻量级，但协程也不能离开线程，它也是在线程中运行的。协程的设计是为了解决多任务并发时的回调问题，通过开启一个协程，在其中运行suspend挂起函数代码，我们可以直接返回结果、流式调用，而不必通过添加回调等待结果返回。
    >
    > 协程的创建包括：
    >
    > 1.通过runBlocking顶层函数创建；
    >
    > 2.通过launch函数创建，launch函数不是顶层函数，需要通过GlobalScope或创建CoroutineScope调用，可以通过coroutineScope.launch(Dispatcher.IO)来切换任务执行的线程，或者Dispatcher.Main切换到主线程。在launch函数中也可以通过withContext(Dispatcher.IO/Dispatcher.Main)来切换线程。
    >
    > 在Android中构建协程可以使用：
    >
    > 1.ViewModel的viewModelScope来开启协程
    >
    > 2.也可以使用Lifecycle中的LifecycleScope开启协程（androidx包下Activity/Fragment实现Lifecycle的）
    >
    > 3.也可以使用liveData构建块用作协程
    >
    > 创建好协程之后，就可以在协程中使用挂起函数，挂起函数也必须在协程中使用。当协程执行到挂起函数时，代码会被挂起，但是不会阻塞住协程所在的线程。当挂起函数返回后，协程中代码会继续执行。挂起函数的本质还是切换线程，执行挂起函数时切换到该函数执行的线程，挂起函数返回后切换回协程所在的线程。挂起函数只能运行在协程中，就能保证协程所在的线程不被阻塞，阻塞的只是协程。

41. HandlerThread原理, 对比单个New Thread的好处，优点以及试用场景

    > 见22题

42. Retrofit+Okhttp+Rxjava在华为的好多手机会OOM是由线程数溢出引起如何解决？

    > 开线程的内存开销包括Java虚拟机内存区的虚拟机栈、程序计数器、本地方法栈，程序计数器内存开销不大且是唯一不会OOM的内存区域，此问题应是虚拟机栈内存溢出了。具体原因还需具体分析。

43. BlockCanary原理

    > BlockCanary原理是通过调用主线程Looper#setMessageLogging，传入自定义的Printer类，计算Handler处理一次消息的时间，如果小于默认配置的3000ms，那BlockCanary会将此Message的信息通过通知栏显示给用户。

44. Android官方MultiDex原理

    > MultiDex是为了解决单个dex方法数65535的限制，在打包的过程中将dex分拆为一个主dex和多个分dex，当app运行时在Application的attactBaseContext方法中，调用MultiDex.install()加载分dex。Dalvik虚拟机加载分dex包，需要将每个dex进行zip压缩，然后进行odex优化，即将dex文件遍历扫描并优化重写，最后通过DexFile.loadDex加载返回DexFile对象并放入ClassLoader的pathList中，这个生成zip和odex文件的耗时是非常严重的。在Android5.0以上默认是支持MultiDex分包的，不需手动加入，并且因为5.0以上使用Art虚拟机，在apk安装过程中会将dex文件预编译为.oat文件提供给Android设备执行，因此也不存在MultiDex加载缓慢的问题。
    >
    > ClassLoader加载的dex文件得到的DexFile对象存放在ClassLoader中DexPathList类型字段的dexElements数组中，MultiDex背后原理就是将分dex对应的DexFile对象存放在扩充的dexElements中，再通过反射替换原数组。
    >
    > 这也是一些热更新方案的原理，通过将更新包的DexFile放在原dexElements的扩充数组的前面，替换旧的dexElements数组，这样加载的将是数组中排列靠前的新类。

45. Java调用kotlin 如何不用companion object{}包裹？

    > companion伴生对象，其中的成员变量和方法在Java中对应的是静态的成员变量和方法

46. Dalvik与ART区别？

    > Dalvik虚拟机在Android5.0以下机型上使用，加载dex优化过的odex文件，JIT（Just In Time）运行代码，动态编译，每次运行代码都需要对odex重新编译。
    >
    > ART虚拟机在Android5.0及以上机型使用，应用安装时会启动dex2oat把dex预编译成ELF文件，AOT（Ahead Of Time）静态编译，运行程序时不再需要重新编译。AOT模式解决了应用启动慢、运行速度慢和耗电问题，但是应用安装过程耗时严重且更占用存储空间。
    >
    > Android7.0开始使用JIT和AOT混合模式，应用安装时不会对dex进行编译，运行时dex文件会先通过解析器后直接执行，与此同时热点函数（Hot Code）会被识别并被JIT编译后储存在jit code cache中并生成了记录热点函数信息的profile文件；当手机进入空闲或充电状态时，系统会扫描App目录下的profile文件并执行AOT编译。这种模式集合JIT和AOT的优点，在应用安装速度加快的同时，运行速度、存储空间、耗电量都得到了优化。

47. Retrofit CreateApi实现原理

    > Retrofit调用createApi后会创建传入接口的动态代理类实例，在InvocationHandler的invoke方法中，通过method对象后通过loadServiceMethod从一个Map缓存中获取的ServiceMethod实例，并调用其invoke方法。ServiceMethod对象中包含Retrofit注解方法的全部信息，如GET/POST请求方式和请求地址、方法上的注解、方法参数的注解、方法返回值类型等。
    >
    > ServiceMethod的实现类是HttpServiceMethod，在HttpServiceMethod的invoke方法会创建一个OkHttpCall对象，并调用到CallAdapter的adapter方法。CallAdapter的实现类是我们创建Retrofit实例时通过Builder的addCallAdapterFactory方法传入，它内部负责创建OkHttp的Call实例并发起请求，最后将返回结果转化为ServiceMethod信息中保存的Method返回类型。返回结果的原始数据转化指定返回类型的数据，是通过ServiceMethod中保存的Convert来转化，这个Convert对象也是我们在创建Retrofit实例时通过Builder的addConvertFactory方法传入。

48. Retrofit 如何实现文件（或图片上传）接口是如何定义的

    > 接口方法使用@MultiPart注解，@POST指定上传地址，file参数注解为@Part、类型为MultipartBody.Part，其余上传字段params通过@PartMap注解、参数类型Map<String,String>。
    >
    > 通过“image/jpeg"创建MediaType，结合File创建RequestBody，再通过RequestBody创建MultipartBody.Part参数。
    >
    > 另外说一下Retrofit的下载监听：下载方法使用注解@Streaming，执行请求方法时通过获取响应流response.body().byteStream()保存到文件中，文件大小为response.body().contentLength()，根据叠加当前写入的流的字节数除以文件大小，可以得到当前下载进度。

49. Android中为什么主线程不会因为Looper.loop()里的死循环卡死？

    > 不会。loop方法内部获取消息是调用MessageQueue的poll方法，而MessageQueue的poll的实现是在native层，利用Linux的poll机制获取Message，当没有剩余的消息可获取时，系统会释放cpu资源，当消息来临时自动唤醒。

50. 解决Android多线程访问SQLite数据库死锁问题？

51. Android多进程解决单个进程内存分配？

    > 说一下进程间的内存分配吧。
    >
    > Android平台在运行时不会浪费可用的内存，总是尝试使用所有可用的空闲内存。例如，系统在应用关闭后都会将它保存在内存中，以便用户快速切换回来，所以Android也没有什么多余的空闲内存。
    >
    > 当内存不足时，有两种主要机制来管理内存：内核交换守护进程（kernel swap daemon，也称kswapd）和低内存终止守护进程（low-memory killer，也称LMK）。
    >
    > Kswapd在设备可用内存不足、低于某一个阈值时变为活跃状态，开始回收内存，当可用内存达到某个上线阈值时，停止回收。很多时候kswapd不能为系统释放足够内存。这种情况下，系统会使用onTrimMemory()通知应用内存不足，应该减少内存占用。如果还不够，内核会通过LMK来终止进程以释放内存。LMK使用oom_adj_score来为正在运行的进程评定优先级，得分越高的进程优先级越低，在清理时也将最先被清理。

52. WeakReference使用场景？java四种引用类型

    > Java的四种引用类型分别是强引用、软引用、弱引用（WeakReference）、虚引用。创建对象后，在堆中为对象分配内存，虚拟机栈中会引用指向堆中对象的地址或对象的句柄，这个引用就是强引用。被强引用引用的对象，GC回收器就认为它是活跃状态，不会回收这部分内存，但有时候这个对象其实已经用不到了，这就造成了内存泄漏。内存泄漏会影响其他对象的可用内存空间，当内存空间不足时则会引起内存溢出，虚拟机抛出OutOfMemory异常。
    >
    > 为解决强引用引用的对象不再使用时，其内存无法回收的问题，Java使用其他三种类型的引用。软引用引用的对象，在虚拟机内存不足时，会回收其引用对象的内存；弱引用引用的对象，GC的时候随时可能回收其内存；虚引用，无法通过它获得对象的实例，只能在对象内存被回收时收到一个回调。
    >
    > WeakReference常用于某个对象需要持有Activity/Fragment实例的情况下，为防止Activity/Fragment内存泄漏，而使用WeakReference持有Activity/Fragment的弱引用。

53. SurfaceView, TextureView区别？

    > SurfaceView和TextureView两者都是继承View，都是在独立的GPU线程中进行绘制和渲染。
    >
    > SurfaceView和Activity的View结构不在同一个图层，无法对其进行平移、旋转、缩放操作；TextureView是在View的图层之下，可以进行平移、旋转、缩放操作。SurfaceView的电池性能是优于TextureView的，Android 7.0强烈推荐使用SurfaceView替代TextureView。
    >
    > SurfaceView的界面位于window界面的z轴下面，当SurfaceView需要显示时，其上层的window会变成透明状态以显示下层的SurfaceView，透明的区域取决于SurfaceView所处的位置。要使用SurfaceView提供的界面，需要调用getHolder()方法，并添加SurfaceHolder.Callback回调，在surfaceCreated和surfaceDestroy方法中监听界面的显示隐藏。需要注意的是，SurfaceHolder的回调方法都是在绘制线程中执行的。

54. Appliction启动过程（App启动过程，需要结合systemServer启动等一起说）

    > 应用启动涉及四个进程：调用者进程（点击桌面图标启动，那么就是Launcher进程）、SystemServer进程（AMS管理Activity的启动）、Zygote进程（fork子进程：共享代码空间、数据空间独立）、新的应用进程。
    >
    > 整体流程是：点击桌面图标启动时，启动请求通过Binder方式发送给SystemServer中的ActivityManagerService，AMS接收到启动请求后检查应用进程是否存在，如果不存在则通过Zygote进程fork子进程，之后回到AMS进程处理处理Intent、Flag信息，创建任务栈、Activity进栈，之后转到新进程，创建ActivityThread对象，调用其main方法，main方法所在的就是主线程，其中还会创建Looper绑定主线程，之后调用loop()方法开启消息循环。

55. apk瘦身

    > 首先要了解apk的结构，它是一个zip压缩文件，解压后包含以下文件和目录：多个classes.dex文件、AndroidManifest.xml文件、resources.arsc文件（包含res/values/下的资源）、META-INF目录（包含CERT.SF和CERT.RSA签名文件以及MANIFEST.MF清单文件）、assets目录（包含应用的资源，可以通过AssetManager检索到这些资源）、res目录（包含未编译到resources.arsc中的资源）、lib目录（包含特定平台子目录的已编译代码，如armeabi-v7a、arm64-v8a目录）
    >
    > 使用Android Studio的插件“Android Size Analyzer”分析apk，瘦身方案从缩减资源数量、大小和减少原生、Java代码方面考虑：通过lint工具发现未使用的资源、代码后移除之；压缩、混淆应用；仅支持特定屏幕密度的设备，如所有图片资源仅使用xxxhdpi的资源；使用Android Studio将现有图片转化为webp文件格式图片等等。

56. IntentService是什么

    > IntentService是一个为了处理异步任务而设计的Service。通过Context.startService启动之后，可以在其onHandleIntent方法中处理Intent信息，此方法运行在异步线程。当异步任务执行完毕之后，会调用stopSelf结束此IntentService，释放内存。
    >
    > 原理是：在IntentService的onCreate方法中，会创建并启动一个HandlerThread线程，通过此异步线程中的Looper对象，创建自定义的Handler子类ServiceHandler。在IntentService的onStart方法中，会将intent包装成一个Message对象，再通过ServiceHandler实例发送到HandlerThread线程处理。而ServiceHandler是重写了Handler的handleMessage方法，在其中调用onHandleIntent处理任务和调用stopSelf结束服务。

57. Retrofit原理

    > 见47题

58. Okhttp原理

    > OKHttpClient类负责管理配置信息、线程调度、请求队列；Request类用来描述请求参数；Response类用来描述响应结果；Call接口负责执行具体请求。
    >
    > OKHttpClient中有一个Dispatcher对象，其内部有维护了一个线程池、三个任务队列（一个队列储存同步任务、一个队列储存正在执行的异步任务、一个队列储存等待中的异步任务）。
    >
    > Call接口中定义了创建请求Request对象的方法、同步/异步执行请求返回Response的方法、以及取消请求等方法。下面分析Call实现类RealCall的同步请求方法execute和异步请求方法enqueue。RealCall#execute方法会将Call对象加入Dispatcher的同步队列中，并调用getResponseWithInterceptorChain方法，该方法会返回请求结果，之后将此Call从同步队列中移除。RealCall#enqueue方法会创建一个AsyncCall对象并将其加入异步等待队列中，之后进行判断能否执行此任务（当前请求的任务数是否小于配置、当前Host的请求数是否小于配置），可以执行的话会将此AsyncCall从等待队列中移出并加入异步执行队列，并放入异步线程中执行，执行过程也是调用getResponseWithInterceptorChain方法获取返回结果，再将结果通过传入的Callback回调回去，最后还会调用Dispatcher的finish方法，从等待队列中取下一个任务。
    >
    > 正如上所说，OKHttp无论是同步请求还是异步请求都是通过Call的实现类RealCall中的getResponseWithInterceptorChain方法获得返回结果的。这个过程是责任链模式，将一次网络请求过程（处理重试与重定向、配置请求头参数、处理缓存配置、连接服务器、执行流操作）交给了不同的拦截器处理。getResponseWithInterceptorChain方法通过创建一个拦截器的ArrayList集合，依次添加自定义拦截器集合、RetryAndFollowUpInterceptor、BridgeInterceptor、CacheInterceptor、ConnectInterceptor、自定义Network拦截器集合、CallServerInterceptor，并将拦截器集合交给Interceptor.Chain的实现类RealInterceptorChain对象的chain方法处理：会从拦截器集合中取出一个拦截器并调用其intercept方法，而intercept方法中又会构建新的Chain对象，并调用拦截器集合中的下一个拦截器的intercept方法。请求是从拦截器集合中从前到后执行，响应则是从拦截器集合中从后到前依次返回。开发过程中可自定义两种拦截器，一种排在所有okhttp定义的拦截器之前，一种是network拦截器排在最后读写流的CallServerInterceptor之前。第一种拦截器拦截到的请求参数是最原始的、响应参数是最完整的；第二种Network拦截器获得的请求参数是最完整的、响应参数是最原始的。使用哪种看业务需求。
    >
    > Ps：缓存原理见30题

59. AIDL

    > 见27题

60. APK打包流程和其内容

    > 1.通过aapt工具打包资源生成R.java文件、aidl工具生成对应java文件
    >
    > 2.编译Java文件生成class文件，再通过dx工具生成classes.dex文件
    >
    > 3.apkbuilder工具将dex文件、编译的资源文件、其他资源文件打包生成apk文件
    >
    > 4.Jarsigner工具签名
    >
    > 5.zipalign工具将apk对齐处理：所有资源文件距离文件起始偏移为4字节整数倍，这样内存映射访问apk文件时速度更快
    >
    > ps 签名过程：通过hash算法计算原始数据的摘要，再通过密钥库中的私钥加密摘要信息得到签名信息，最后将签名信息写入签名区。校验签名：提取apk文件摘要与公钥对签名区的数字签名进行解密得到的原始摘要进行比较。keystore签名文件中包含签名和校验过程需要的私钥、公钥和数字证书，其中公钥、私钥是非对称加密算法密钥；数字证书是由身份认证机构颁发的，可以保证密钥可靠性。

61. Asynctask原理及不足

    > AsyncTask是Android系统提供的用于执行异步任务的类
    >
    > 其特点如下：
    >
    > 1.创建了三个泛型Params、Progress、Result，分别代表了三个需要重写的方法doInBackground、onProgressUpdate、onPostExecute的形参类型，另外还定义了一个onPreExecute方法；
    >
    > 2.封装了两个线程池，用户可以选择异步任务是串行执行或并行执行；
    >
    > 3.与主线程交互：doInBackground执行异步任务时，可以通过publishProgress方法发送参数，在主线程的onProgressUpdate方法中将收到回调，doInBackground的返回结果也将在主线程的onPostExecute中作为形参返回，其原理是AsyncTask内部维持着一个主线程的Handler。AsyncTask内部还有一个没有对Message作任何处理的Handle对象，当传入给AsyncTask的Looper不是主线程的Looper时，将通过这个无实现的Handler发送消息，onProgressUpdate和onPostExecute方法都不会被调用到。因此，AsyncTask只能作为异步方法与主线程交互情形时使用；
    >
    > 其原理如下：
    >
    > 1.构造方法：无参有参数Handler的都会调用有参数Looper的构造，在其中创建三个成员变量：mHandler、mWorker、mFuture。传入Looper如果是否为主线程Looper，mHandler则是主线程中有处理消息的Handler实现类，否则是一个无任何消息处理的异步线程Handler。mWorker是一个WorkerRunnable实现类，在call方法中调用doInBackground方法、在finally块中调用postResult方法将返回结果通过Handler发送到主线程中的onPostExecute方法中执行；mFuture是一个FutureTask的对象，传入mWorker对象，其中定义了一些取消请求的方法。
    >
    > 2.execute方法：内部调用的是executeOnExecutor(sDefaultExecutor,params)方法，sDefaultExecutor是一个串行的线程池，因此execute默认是串行执行。executeOnExecutor方法内部首先调用onPreExecute方法，是我们可以实现在异步任务执行之前调用的方法，可以作一些主线程的初始化操作。之后将参数params交给mWork的mParams参数，调用传入的线程池执行mFuture任务，mWorker中的call方法也就会在异步线程中执行。
    >
    > 不足：
    >
    > 1.AsyncTask设计初衷就是为了异步任务与主线程交互时使用，也只支持在主线程使用；
    >
    > 2.AsyncTask如果是Activity的非静态内部类，那么当Activity销毁而有任务正在执行时，会造成Activity内存泄漏，解决办法是AsyncTask需要是静态的并持有Activity的弱引用；
    >
    > 2.AsyncTask并不具备Activity生命周期感知功能，如果Activity销毁前没有调用cancel方法取消任务，任务仍将继续执行，并有可能调用销毁了的Activity更新UI的方法造成crash。解决办法是在Activity销毁时调用cancel方法取消任务，并在AsyncTask中所有更新UI的操作前，调用isCanceled方法判断任务是否已取消。

62. 如何实现一个网络框架

    > 首先明确一点，此处实现的网络框架是在okhttp或httpurlconnection基础上封装的网络框架，如果是去实现一个底层网络框架，难度太大不在考虑范围。在okhttp的基础上再去封装一层，基本目的是为了做接口隔离，好处是当需要替换okhttp时，我们能轻松替换。
    >
    > 主要工作：封装处理网络请求的4个要素：请求方式、请求地址、请求参数、请求回调。
    >
    > 另外需要考虑设计模式，按照面向对象的六大原则来：
    >
    > 单一职责原则（一个类只实现单一的功能）、
    >
    > 开闭原则（类、模块、函数对扩展开放对修改封闭）、
    >
    > 里氏替换原则（所有引用基类的地方都能使用其子类对象）、
    >
    > 依赖倒置原则（模块之间不依赖直接实现，要依赖接口或抽象类）、
    >
    > 接口隔离原则（不依赖不需要的接口）、
    >
    > 迪米特原则（一个对象只与直接关联对象交互）。

63. 理解Window和WindowManager

    > ~~Window表示窗口的概念，每一个Window都对应一个View和一个ViewRootImpl，Window和View之间通过ViewRootImpl建立联系。Window不是实际存在，它是以View的形式存在的。WindowManager对Window进行管理，包含addView、updateViewLayout、removeView这三个方法。~~
    >
    > ~~Window的实现是PhoneWindow，Activity中调用setContentView的具体实现其实是在PhoneWindow中，通过window对应的decorView找到“R.id.content”对应的控件，将布局填充到这个控件中。~~

64. Android IPC机制，描述一次跨进程通讯

    > 跨进程通信无处不在，简单的一个startActivity都是涉及到跨进程的

65. 动画有哪几类，各有什么特点

    > 属性动画、视图动画（帧动画、补间动画）
    >
    > 帧动画是按照一定显示顺序来显示图片；补间动画则是对视图进行旋转、淡入淡出、移动、拉伸等转换；属性动画是在设定的时间内修改对象的属性值。补间动画只是对视图进行一些转换，并没有真正改变视图的属性，也不能用与颜色值的变化；而属性动画则是实实在在修改了对象的属性，也可用于修改颜色值。
    >
    > Android还根据动画应用场景提供了其他Api：
    >
    > 1.使用动画显示或隐藏视图：创建淡入淡出的属性动画；创建卡片翻转动画；创建圆形揭露动画
    >
    > 2.使用动画移动视图：利用ObjectAnimator创建属性动画，也可以使用PathInterpotor添加曲线运行
    >
    > 3.基于物理特性的动画：Fling动画（滑出后阻尼停止）、弹簧动画
    >
    > 4.使用Transition为布局变化添加过渡效果

66. Intent可以传递哪些数据类型

    > Intent传递的是Bundle数据，Bundle支持基本数据类型、Parcelable、Serializable以及它们对应的ArrayList、Array类型的数据。

67. android四大组件的加载过程，请详细介绍下

    > ~~Activity~~
    >
    > ~~启动过程：调用startActivity之后启动过程交给了AMS处理（创建进程、处理进栈、创建ActivityThread并调用main方法），如果进程、栈存在，那么通过IPC通知ActivityThread的内部类ApplicationThread执行创建Activity的任务，通过ActivityThread中的Handler对象mH，执行handleLaunchActivity、performLaunchActivity等操作，使用classloader加载Activity、反射创建其对象，创建Application、创建contextImpl对象赋值上下文、创建ViewRootImpl对象连接Window和DecorView，之后再通过mH发送执行Activity生命周期方法的Message。~~
    >
    > ~~销毁过程：通过mH发送执行销毁过程生命周期方法的Message；关闭Window、移除DecorView；调用ContextImpl执行清理操作，通知AMS清理Activity绑定的服务与广播；最后通知AMS此Activity销毁了。~~
    >
    > （这类问题我太厌倦了）

68. 介绍下AMS，PMS，installed等核心服务

69. Android组件化

    > 开发过程中为了代码重用和业务解耦，当我们把单一的功能放在不同的module中，各个module相互独立、没有依赖关系并可以单独运行调试，那这种划分则是组件化。
    >
    > 1.组件module的gradle文件中，通过一个布尔变量来控制module类型，是application或者是library，如果是application还需要指定ApplicationId，sourceSet也需要指定application和library状态下不同的AndroidManifest文件。这样此组件既可以继承调试又可以单独调试，仅需要一个布尔变量来控制。
    >
    > 2.各组件没有依赖关系但却需要互相调用方法、传递数据，比如分享组件需要调用登陆组件中的方法获取登陆状态才能继续分享、获取其他组件中的Fragment等场景。基于接口编程可以解决此问题，在各个组件之下，创建一个baseModule被所有组件依赖，创建接口定义某个组件需要暴露的方法，再创建接口实例的生产工厂。接口的具体实现在上层组件中，创建一个组件初始化的方法在程序启动时调用，在其中将需要暴露的对象赋值给baseModule中的接口实例工厂，这样其他组件就可以通过baseModule中获取到接口实例来调用其他组件的方法。需要注意的是，当不需要加载某个组件时，该组件的初始化方法就不需要调用，因此通过反射来调用组件中的初始化方法，并捕获加载没有依赖的组件中的class导致的异常。
    >
    > 3.路由实现组件界面跳转。在组件中Activity上通过注解指定path，编译时通过apt生成路由表，其他组件通过baseModule中定义的跳转Activity方法，根据传入的path找到路由表中的Activity实现跳转与传参。

70. 对gradle、proguard、aapt的了解

    > **gradle**是一个项目自动化构建工具，构建就是根据输入信息执行一系列操作最后得到产出物。Gradle就是帮助管理项目中的差异、依赖、编译、打包、部署的。
    >
    > Android Gradle项目一般包含如下文件：
    >
    > /gradle/wrapper/gradle-wrapper.properties：distributionUrl配置gradle版本；
    >
    > build.gradle：buildscript中dependencies配置gradle插件版本；
    >
    > gradle.propeties：配置全局gradle设置；
    >
    > setting.gradle：配置工程树；
    >
    > local.properties：配置sdk、ndk的路径；
    >
    > gradlew和gradlew.bat：分别为mac和window系统执行Android项目中的构建任务。
    >
    > **proguard**是Android的混淆工具，位于/sdk/tools/proguard目录下，我们可以在module/build.gradle文件中buildTypes/${build variant}下配置minifyEnable true开启混淆，通过proguardFiles指定混淆配置文件，一般制定getDefaultProguardFile('proguard-android.txt')为/sdk/tools/proguard下的配置文件，另外还需制定自定义的配置文件（用于配置一些需要保持不混淆的JavaBean）。构建后会在<module-name>/build/outputs/mapping/${build variant}/目录下生成dump.txt文件说明apk中所有类文件的内部结构；mapping.txt文件包含原始对混淆过的类、方法、字段名称之间的映射关系；seeds.txt文件列出未进行混淆的类和成员；usage.txt列出apk移除的代码。
    >
    > **aapt**是指Android资源打包工具（Android Asset Packaging Tool），在Android Gradle Plugin3.0.0开始使用aapt2，位于$ANDROID_HOME/platforms/$VERSION目录下。aapt2流程包括编译和链接两步。编译将资源文件生成.flat格式的临时二进制文件，链接将所有临时二进制文件打包到一个没有dex、没有签名的apk中，并且会生成R.java文件。此apk包含所有资源，以及一个resources.arsc文件。resource.arsc文件是一个资源映射表，其中记录了id值、name以及针对屏幕大小、像素密度等对应信息，根据资源id和具体场景信息（比如分辨率），就可以找到资源的具体地址。

71. dex结构、class结构、apk结构

    > apk结构
    >
    > dex文件：可执行文件
    >
    > res文件夹：存放应用程序的资源
    >
    > assets文件夹：存放打包到apk中的静态文件
    >
    > lib文件夹：存放native库
    >
    > META-INF文件夹：存放应用程序签名和证书
    >
    > AndroidManifest.xml文件：清单文件
    >
    > resources.arsc文件：资源配置文件
    >
    > 其他：如kotlin文件夹（kotlin_metadata文件）、okhttp3文件夹
    >
    > **待补充**
    >
    > class结构
    >
    > dex结构

72. 跨平台开发技术有哪些？你了解哪些（weex、RN、flutter）

73. ContextWrapper和Activity这些Context有什么区别。

    > Activity继承ContextThemeWrapper，ContextThemeWrapper继承ContextWrapper，ContextWrapper继承Context。ContextWrapper中有一个Context的成员mBase，在Activity创建的时候，也就是ActivityThread#performLaunchActivity方法中会创建一个ContextImpl对象（ContextImpl也是继承Context），并通过调用Activity#attach方法、Activity#attachBaseContext方法将ContextImpl对象赋值给mBase。典型的装饰模式，Activity中继承Context的方法具体实现在ContextImpl类中。

74. Activity、 Window、 View三 者 的 差 别 ， fragment的 特 点 ?

    > Activity显示依靠Window，Window对应的是一个View。在Activity中调用setContentView传入View后，会交给Window的子类PhoneWindow处理，PhoneWindow通过ViewRootImpl管理着一个DecorView，传入的View会放到DecorView的视图树中。Fragment可以看作是对Activity中一个View，但是具备与Activity联动的生命周期回调。

75. Handler、 Thread和 HandlerThread的 差 别 

    > 见22题

76. 低版本SDK实现高版本api

    > 首先弄清在build.gradle中配置的compileSdkVersion、minSdkVersion、targetSdkVersion的区别，当minSdkVersion小于某个api支持的版本时，做运行时版本判断。

77. 编译安卓系统

78. Android View刷新机制

79. 优化自定义view

80. fragment生命周期

81. ContentProvider

82. SharedPreferences使用中遇到过哪些问题？如何解决

83. MVP 及 MVP 怎么解决内存泄漏

84. selector 为什么能够切换背景，原理是什么

85. Fragment 的 commitAllowStateLoss 方法

86. 单击事件和双击事件哪个先触发

87. 不考虑具体页面，怎么从根本上优化界面卡顿

88. SurfaceFlinger、VSYNC

89. 说一下 ActivityManagerService、ActivityManagerNative 等几个类的区别

90. OkHttp底层网络请求实现，socket还是URLConnection？

91. ViewPager如何判断左右滑动？

92. Include、Merge、ViewStub的作用

93. 假设ListView中有10W个条项，那内存中会缓存10W个吗？

94. ListView和RecyclerView的区别？

95. animation和animator的用法,概述实现原理

96. app闪退的原因有哪些?每种情况简述分析过程

97. 如何加载ndk库?如何在jni中注册native函数,有几种注册方式?

98. ams是怎么找到启动的那个activity的?

99. 在oncreate里面可以得到view的宽高吗?

100. view的getwidth和getmesurewidth有啥区别?

101. assets与res/raw的区别?

102. android常用布局及排版效率

103. 补间动画常见的效果?有哪几个常见的插入器?

104. 请结束android.mk的作用,并试写一个android.mk文件(包含一个.c源文件即可)

105. ART和Dalvik的区别？在gc上有何改进?

106. bitmap有几种格式,分别占多少字节

107. ConstaintLayout

108. 一个app启动页另开一个进程,启动页10s后启动mainactivity,请问5s的时候有几个进程?

109. 好几万条短信,滑动卡顿怎么解决?

110. 计算viewgroup的层级,递归实现和非递归实现

111. zxing有过优化提高识别率吗?

112. 补间动画click事件还在原位怎么解决?

113. 隔代数据库升级

114. Arraylist里面可以不可以new一个T泛型的数组?

115. startActivityForResult是哪个类的方法,在什么情况下使用,如果在Adapter中使用应该如何解耦

116. ui绘制流程和activity生命周期有什么关系，或者ui开始绘制的时机到底在什么时候？

117. 你对网络请求做过哪些优化呢

118. Okhttp有什么优秀的设计模式?builder模式有什么好处?责任链模式有什么好处?

119. a-b-c界面,其中b是single instance 的,那么c界面点back返回a界面,为什么?怎么管理栈的?

120. viewpager嵌套滑动冲突怎么解决?

121. arraymap和hashmap的区别?

122. viewpager嵌套滑动冲突怎么解决?

123. svg动画

     > Android 5.0开始引入了VectorDrawable和AnimatedVectorDrawable，以支持svg图片的显示。VectorDrawable定义静态可绘制对象，在xml文件中定义树形层次结构、由path和group组成。AnimatedVectorDrawable会为矢量图形的属性添加动画，可以将添加动画效果之后的矢量图形定义在三个单独的资源文件中。对于Android 5.0以前的版本，提供兼容的类VectorDrawableCompat和AnimatedVectorDrawableCompat。

124. 属性动画画一个抛物线怎么弄?

125. 为了适配多分辨率,引入什么开源框架?

126. 布局怎么做到每行的文字左右对齐?

     > [不到100行代码实现左右对齐TextView](https://www.jianshu.com/p/7241ed34346a)

127. framework加载activity的流程

     > 见67题

128. 自己写一个应用,包名就叫android行不行,为什么?

     > 不行，IDE提示至少要一个符号“.”
     >
     > 如果包名为"com.android"，是可以的

129. 主线程looper如果没有消息,就会阻塞在那,为什么不会ANR?

     > ANR指的是系统无法响应用户触摸事件和绘制操作时抛出的异常。主线程looper虽然阻塞，但并不会影响用户事件的处理，反而正是looper处理了用户的触摸事件。



## Java
1. Java多线程
2. hashmap的实现原理，如何解决hash冲突的
3. 静态方法是否能被重写
4. 进程与线程区别
5. 单例
6. 常用设计模式和设计的六大原则
7. 线程同步
8. GC，包括GC算法（引用计数、可达性分析等），还有分代回收等等
9. concurrentHashmap原理，原子类。
10. 集合框架，list，map，set都有哪些具体的实现类，区别都是什么
11. volatile原理
12. Interger中的128(-128~127), 常量池的概念
13. 线程池
14. java中的泛型，泛型擦除以及相关的概念
15. 多进程间的通讯
16. map接口下都有什么子类->hashmap和hashtable区别->hashmap实现原理->怎么解决hash冲突->是否了解concurrentHashmap->concurrentHashmap实现原理->volatile实现原理（concurrentHashmap读是不加锁的，使用到了volatile）
17. ThreadLocal原理
18. 多线程断点续传原理
19. classloader泛型是什么以及在项目中的应用
20. arraylist和linkedlist的区别，以及应用场景
21. 内部类和静态内部类和匿名内部类作用，以及项目中的应用
22. java注解以及Android中的应用&APT
23. String和StringBuffer、StringBuilder的区别
24. String s = “123”;这个语句有几个对象产生
25. Exception和RuntimeException的区别，作用又是什么？
26. java异常体系知道吗？error和exception有什么区别？
27. reader和inputstream区别
28. char型变量中能不能存贮一个中文汉字?为什么?
29. java编译时和运行时有什么区别？
30. 堆内存，栈内存理解，栈如何转换成堆？内存泄漏是发生在堆内存还是栈内存？为什么？
31. 如何实现打印指定阻塞线程的方法名？
32. LinkedHashMap与HashMap区别？
33. String a=“A” 与 String a = new String（“A”); 区别？分别存储在哪个区域
34. 代理模式和装饰器模式区别？
35. 类的加载过程，Person person = new Person();为例进行说明。
36. java泛型类型擦除发生在什么时候，通配符有什么需要注意的。
37. 操作系统如何管理内存的
38. HashTable的实现被废弃的原因
39. java的线程如何实现
40. CopyOnWriteArrayList怎样实现
41. 接口的意义
42. java的集合和它们之间的继承关系
43. java虚拟机的特性
44. 进程和线程的区别, 系统什么时候会在用户态和内核态中切换
45. Java中 ==和 equals的 区 别 ， equals和 hashCode的 区 别
46. java 状态机
47. java中int char long各占多少字节数
48. Java多态的理解
49. 什么导致线程阻塞
50. 抽象类和接口区别
51. ArrayMap VS HashMap
52. 线程和协程，为什么协程比线程效率高
53. Eden 和 Survivor 的比例和回收规
54. 怎么判断对象是否要进入老年代
55. 新生代为什么用复制算法
56. 什么是迭代器失效？
57. Hash一致算法？
58. java的内存分区？
59. Jvm中的常见的垃圾回收器？
60. CMS和G1了解吗？
61. CMS解决什么问题，说一下回收的过程？
62. 线程池由哪些组件组成？
63. 线程的启动和终止？
64. 有哪些线程池，分别怎么使用？拒绝策略有哪些？
65. 什么时候多线程会发生死锁，写一个例子？
66. AtomicInteger内存模型
67. 用四个线程计算数组和
68. 遍历HashMap的原理?
69. 写生产者消费者模式,不可用syncronized
70. 原子类的了解



## 算法
1. 链表反转的算法题
2. 写出所有数组的子序列
3. 数据结构，搜索二叉树的一些特性，平衡二叉树
4. 二分查找
5. 快排
6. 回型打印二维数组
7. 字符串反转，讨论复杂度。
8. 给定一个int型 n，输出1~n的字符串例如 n = 4 输出“1 2 3 4”
9. 输出所有的笛卡尔积组合
10. 最长上升子序列
11. 给出二叉树和一个值，找出所有和为这个值的路径；{1,3}{3,6}{3,4}{6,8}区间去重，最少去掉几个集合，可以让这个集合没有交集。
12. 将一个字符串转换成int型数字，考虑 错误输入，溢出，正负值等一些条件
13. m * n的矩阵，能形成几个正方形（2 * 2能形成1个正方形，2 * 3 2个，3 * 3 6个）
14. 堆排序
15. 链表的各种操作：判断成环、判断相交、合并链表、倒数K个节点、寻找成环节点
16. 二叉树、红黑树、B树定义以及时间复杂度计算方式
17. 动态规划、贪心算法、简单的图论
18. 常见的排序算法时间复杂度
19. 依次打印二叉树每层最左边的结点
20. 跳台阶问题
21. 求两个链表的交点
22. 判断二叉树是否左右对称（只考虑结构对称，不考虑值）
23. 猴子偷桃问题代码实现
24. 如何判断一个字符串是回文字符串



## 网络
1. 3次握手和4次挥手的原因，以及为什么需要这样做
2. http中的同步和异步
3. 网络五层结构
4. https相关，如何验证证书的合法性，https中哪里用了对称加密，哪里用了非对称加密，两者的区别
5. 知道socket吗？和Websocket有什么区别？
6. https握手过程，如何实现数据加密？客户端如何保证安全实现双重证书校验？请你设计一个登录功能，需要注意哪些安全问题?
7. 浏览器输入地址到返回结果发生了什么
8. 如何设计在 UDP 上层保证 UDP 的可靠性传输
9. TCP和UDP协议的区别
10. 描述一次网络请求的流程

