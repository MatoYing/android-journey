








>fff
>eed

充电速度
![[../../../assets/Pasted image 20260831180127.png]]
































#### 什么时候用静态变量、静态方法
用在全局唯一、无状态的场景，比较常见的场景：
- 常量
- 工具类方法
- 单例模式
###### 比如 Fragment 里面的东西为啥不用 static
首先，这些类里的 Tag 都是用 static 声明的，因为它和具体某个 Fragment 实例无关，所有的对象都有这个值，是共用的，所以声明为 static。
一方面，假设你有两个 Fragment 实例，一个显示用户A，一个显示用户B，这是有状态的，肯定不能 static。
另一方面，那生命周期完全没有意义了，static 只有在对应 Class 释放时才会被释放，而后者这有进程结束时才会被释放。这样会导致内存泄漏。
###### 常量、工具类方法这些一直不释放不算内存泄漏吗？
内存泄漏的核心定义是：程序中已动态分配的堆内存，由于某种原因，程序**无法释放**或**不再需要**这块内存，但却无法被垃圾回收器回收，导致内存被白白占用，最终可能引发内存溢出。
但对于 static，是**符合预期**的行为。程序员在声明一个 `static Map` 时，就是希望它作为一个全局缓存，在整个程序运行期间都存在。它的存在是“必需的”，而不是“无用的、无法释放的垃圾”。
但是，误用 static 变量是造成 Java 内存泄漏最常见的原因之一，比如你声明了一个 `static Map`，你一直往里面 `add` 东西，有些东西其实已经不用了，但你还不删除，这就会造成内存泄漏。
###### 那这样不占用内存吗？
设置为 static 恰恰是为了更高效、更合理地使用内存，而不是为了避免占用内存。
###### 好处
有两个好处：
- 语义正确性，表达“属于类而非实例”。比如常量 `Math.PI`，它的值是固定不变的，不需要创建 `Math` 对象就能获取，让每个 `Math` 对象都存一份 `PI` 是毫无意义的；比如工具方法 `Arrays.sort()`，这些方法执行的操作不依赖于任何对象实例的状态（即不访问非 static 的实例变量），它们只依赖于传入的参数。因此，不需要创建对象来调用它们，直接通过类名调用最为合理。
- 调用便利性：无需实例化，直接调用。通过类名直接访问，代码非常简洁清晰，避免了繁琐且无意义的`new`操作。



**内存泄漏**：
在程序运行期间，**不再被使用的内存无法被回收**，导致内存占用不断增加，像是使用 static 修饰的集合、**未关闭文件流**等。



#### 方法提供内提供至 null 避免内存泄漏的场景
> 不是很明白，日后再看
public class TabNavigationAction extends VehicleAction {  
​  
    private NavigationListener mNavigationListener;  
    private String mValue;  
​  
    public void setNavigationListener(@NonNull final NavigationListener navigationListener) {  
        mNavigationListener = navigationListener;  
        if (mKeyStr != null) {  
            mNavigationListener.navigate(mValue, mValue);  
        }  
    }  
​  
    public void deleteListener() {  
        mNavigationListener = null;  
        mValue = null;  
    }  
​  
    public interface NavigationListener {  
        public String navigate(String key, String value);  
    }  
}  
​  
​  
public class IntelligentDeviceViewModel extends BaseViewModel {  
    public void registerNavigationListener(final NavigationHandleListener handleListener) {  
        final TabNavigationAction action = getNavigationAction();  
        if (action != null) {  
            action.setNavigationListener(handleListener::navigateHandle);  
        }  
    }  
​  
    // 可以看作等同于NavigationListener  
    public interface NavigationHandleListener {  
        String navigateHandle(@NonNull String key, String value);  
    }  
​  
​  
    private void unRegisterNavigationListener() {  
        final TabNavigationAction action = getNavigationAction();  
        if (action != null) {  
            action.deleteListener();  
        }  
        LogUtils.getInstance().d(TAG, "unRegisterNavigationListener");  
    }  
​  
​  
    @Override  
    public void onCleared() {  
        LogUtils.getInstance().d(TAG, "onCleared");  
        super.onCleared();  
        mDisposables.clear();  
        mRegisterPowerObserUseCase.clear();  
        mOffOperationHandler.removeCallbacksAndMessages(null);  
​  
        unRegisterNavigationListener();  
    }  
}  
​  
​  
public class IntelligentDeviceFragment extends BaseThemeFragment<IntelligentDeviceViewModel> {  
    // 内部类会引用外部类，也就是IntelligentDeviceFragment  
    getViewModel().registerNavigationListener((key, value) -> {  
         LogUtils.getInstance().d(TAG, "页面导航 key = " + key + ", 处理界面--->" + value);  
         String reply = Header.Reply.UN_SUPPORT;  
         if (TextUtils.equals(key, ConstantValue.FUNC_TYPE_SMART_DEVICES)) {  
             // 这里引用了IntelligentDeviceFragment的元素  
             reply = handleVrCommand(mFridgeSwitchLayout);  
         } else if (TextUtils.equals(key, ConstantValue.FUNC_TYPE_SMART_DEVICES_MODE)) {  
             reply = handleVrCommand(mFridgeRecyclerView);  
         }  
         return reply;  
    });  
}

IntelligentDeviceFragment → “IntelligentDeviceViewModel” → TabNavigationAction → NavigationHandleListener  
        ↑                                                                                 ↓   
        ---------------------------------------------------------------------------------->  
    
IntelligentDeviceViewModel → TabNavigationAction → NavigationHandleListener → IntelligentDeviceFragment  
    
    
ViewModel (存活)    
  ↓    
TabNavigationAction    
  ↓    
NavigationListener (lambda)    
  ↓    
Fragment (被 lambda 引用)    
  ↓    
ViewModel (Fragment 持有 ViewModel)    
    
    
    
可能：  
GC Roots  
  ↓  
ActivityThread  
  ↓  
Activity  
  ↓  
FragmentManager  
  ↓  
Fragment  
  ↓  
ViewModelStore（Fragment 的成员）  
  ↓  
ViewModel（持有 lambda）  
  ↓  
lambda（捕获 Fragment）形成闭环

为什么要提供 `deleteListener()`？

因为这里这里有一个循环引用，不受冻至 null，就无法被 GC 回收。

我在想当 Fragment 或 ViewModel 生命周期结束时，这个循环引用不就打破了吗？

当 Fragment 生命周期已经结束，也就是“上面”已经没有东西引用它了，按理应该被 GC 回收，但这里的 listener 还持有对 Fragment 的强引用，GC 可能通过别的路径比如 ViewModel 最终到达 Fragment ，那么 GC 就认为 Fragment 仍“可达”，不能回收，这就导致了内存泄漏。（别去想了）