#### 基本用法

1）引入依赖

```groovy
dependencies {
    implementation 'androidx.recyclerview:recyclerview:1.4.0'
}
```

2）添加 RecyclerView 到布局

activity_main.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

3）Item 布局

item_news.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp">

    <TextView
        android:id="@+id/titleTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="18sp"
        android:textStyle="bold"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"/>

    <TextView
        android:id="@+id/contentTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:textSize="14sp"
        app:layout_constraintTop_toBottomOf="@id/titleTextView"
        app:layout_constraintStart_toStartOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

4）数据模型

```java
public class News {
    public String title;
    public String content;

    public News(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

5）自定义 Adapter + ViewHolder

+ Adapter：负责创建 ViewHolder 和绑定数据
+ ViewHolder：持有 View 引用，避免重复 findViewById
+ `onCreateViewHolder()`：创建 View（复用前调用）
+ `onBindViewHolder()`：绑定数据（每次滑动都会调用）

```java
public class NewsAdapter extends RecyclerView.Adapter<NewsAdapter.NewsViewHolder> {

    private List<News> newsList;

    // 构造函数传入数据
    public NewsAdapter(List<News> newsList) {
        this.newsList = newsList;
    }

    // 1.创建ViewHolder（只创建视图，不绑定数据），之后滑动如果可以复用，不会创建新的
    @NonNull
    @Override
    public NewsViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_news, parent, false);
        return new NewsViewHolder(view);
    }

    // 2.绑定数据到ViewHolder，之后滑动每次重新显示都会调用
    @Override
    public void onBindViewHolder(@NonNull NewsViewHolder holder, int position) {
        News news = newsList.get(position);
        holder.titleTextView.setText(news.title);
        holder.contentTextView.setText(news.content);
    }

    // 3.返回数据总数
    @Override
    public int getItemCount() {
        return newsList.size();
    }

    // ViewHolder内部类：负责持有View引用
    static class NewsViewHolder extends RecyclerView.ViewHolder {
        TextView titleTextView;
        TextView contentTextView;

        public NewsViewHolder(@NonNull View itemView) {
            super(itemView);
            titleTextView = itemView.findViewById(R.id.titleTextView);
            contentTextView = itemView.findViewById(R.id.contentTextView);
        }
    }
}
```

6）Activity 中使用

```java
public class MainActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private NewsAdapter adapter;
    private List<News> newsList = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        recyclerView = findViewById(R.id.recyclerView);

        // 生成测试数据
        for (int i = 0; i < 50; i++) {
            newsList.add(new News("标题 " + i, "内容 " + i));
        }

        // 创建 Adapter 并传入数据
        adapter = new NewsAdapter(newsList);
        recyclerView.setAdapter(adapter);

        // 设置布局管理器（必须设置，否则不显示）
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
    }
}
```

7）数据更新

全量刷新（不会带默认动画）：

```java
// 修改数据后，通知Adapter刷新整个列表
newsList.clear();
newsList.addAll(newDataList);
adapter.notifyDataSetChanged();
```

局部刷新（带默认动画）：

```java
// 插入一项
News newNews = new News("新标题", "新内容");
newsList.add(2, newNews);
adapter.notifyItemInserted(2);

// 删除一项
newsList.remove(3);
adapter.notifyItemRemoved(3);

// 更新一项
newsList.set(1, updatedNews);
adapter.notifyItemChanged(1);

// 批量操作
adapter.notifyItemRangeInserted(start, count);
adapter.notifyItemRangeRemoved(start, count);
adapter.notifyItemRangeChanged(start, count);
```

8）添加点击事件

在 `onBindViewHolder()` 中设置：

```java
@Override
public void onBindViewHolder(@NonNull NewsViewHolder holder, int position) {
    News news = newsList.get(position);
    holder.titleTextView.setText(news.title);
    holder.contentTextView.setText(news.content);

    // 设置点击监听
    holder.itemView.setOnClickListener(v -> {
        int pos = holder.getBindingAdapterPosition();
        if (pos != RecyclerView.NO_POSITION) {
            Toast.makeText(v.getContext(), 
                "点击: " + newsList.get(pos).title, 
                Toast.LENGTH_SHORT).show();
        }
    });
}
```

9）布局管理器

```java
// 线性列表（垂直）
recyclerView.setLayoutManager(new LinearLayoutManager(this));

// 线性列表（水平）
recyclerView.setLayoutManager(
    new LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false));

// 网格布局（2列）
recyclerView.setLayoutManager(new GridLayoutManager(this, 2));

// 瀑布流布局（2列，垂直）
recyclerView.setLayoutManager(
    new StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL));
```

10）添加分隔线

```java
recyclerView.addItemDecoration(
    new DividerItemDecoration(this, DividerItemDecoration.VERTICAL)
);
```

11）Item 动画

局部刷新方法会自动触发默认动画，全量刷新不会。

```java
// 使用默认动画
DefaultItemAnimator animator = new DefaultItemAnimator();
animator.setAddDuration(300);
animator.setRemoveDuration(300);
recyclerView.setItemAnimator(animator);

// 关闭动画
recyclerView.setItemAnimator(null);
```

12）总结

完整代码结构：

```
MainActivity.java    → 初始化RV、设置Adapter、LayoutManager
NewsAdapter.java     → Adapter + ViewHolder内部类
News.java            → 数据模型
activity_main.xml    → 主布局（包含RecyclerView）
item_news.xml        → Item布局
```

关键点总结：

```
setAdapter()：绑定Adapter
setLayoutManager()：设置布局方式（必须）
onCreateViewHolder()：创建View
onBindViewHolder()：绑定数据
notifyItemXxx()：局部刷新 + 动画
notifyDataSetChanged()：全量刷新（无动画）
```

#### DiffUtil - ListAdapter 异步方式

DiffUtil 是 Android 提供的差分算法工具，通过自动对比新旧数据源的差异，向 RecyclerView 精准派发局部刷新指令，其底层本质仍是调用 `notifyItemInserted()`、`notifyItemChanged()` 等局部刷新方法，既免去了手动计算位置的繁琐与风险，又避免了全量重绘 `notifyDataSetChanged()` 的性能损耗。

1）数据模型添加唯一 id

```java
public class News {
    public long id;      // 唯一标识，用于判断是否是同一个item（叫什么都行，和areItemsTheSame()里一致就行）
    public String title;
    public String content;

    public News(long id, String title, String content) {
        this.id = id;
        this.title = title;
        this.content = content;
    }
}
```

2）定义 DiffCallback

新建 NewsDiffCallback.java，告诉 RV 怎么比较两个 News：

```java
public class NewsDiffCallback extends DiffUtil.ItemCallback<News> {

    // 是否是同一个item（比较id）
    @Override
    public boolean areItemsTheSame(@NonNull News oldItem, @NonNull News newItem) {
        return oldItem.id == newItem.id;
    }

    // 内容是否相同（比较具体字段）
    @Override
    public boolean areContentsTheSame(@NonNull News oldItem, @NonNull News newItem) {
        return oldItem.title.equals(newItem.title)
            && oldItem.content.equals(newItem.content);
    }
}
```

3）Adapter 继承 ListAdapter

这是官方最推荐的方式，与普通 Adapter 的区别：

- 不需要自己维护 `newsList`。
- 不需要写 `getItemCount()`。
- 用 `getItem(position)` 获取数据。
- `submitList()` 在后台线程异步计算 Diff，不卡主线程。

```java
public class NewsAdapter extends ListAdapter<News, NewsAdapter.NewsViewHolder> {

    // 构造函数传入DiffCallback
    public NewsAdapter() {
        super(new NewsDiffCallback());
    }

    // 创建ViewHolder
    @NonNull
    @Override
    public NewsViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_news, parent, false);
        return new NewsViewHolder(view);
    }

    // 绑定数据
    @Override
    public void onBindViewHolder(@NonNull NewsViewHolder holder, int position) {
        News news = getItem(position);  // 用getItem()获取数据
        holder.titleTextView.setText(news.title);
        holder.contentTextView.setText(news.content);
    }

    // 不需要自己写getItemCount()，ListAdapter已实现

    static class NewsViewHolder extends RecyclerView.ViewHolder {
        TextView titleTextView;
        TextView contentTextView;

        public NewsViewHolder(@NonNull View itemView) {
            super(itemView);
            titleTextView = itemView.findViewById(R.id.titleTextView);
            contentTextView = itemView.findViewById(R.id.contentTextView);
        }
    }
}
```

4）Activity 中使用

```java
public class MainActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private NewsAdapter adapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        recyclerView = findViewById(R.id.recyclerView);
        adapter = new NewsAdapter();  // 不需要传数据
        recyclerView.setAdapter(adapter);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        // 加载初始数据
        loadNewsList();
    }

    private void loadNewsList() {
        List<News> newsList = new ArrayList<>();
        for (int i = 0; i < 50; i++) {
            newsList.add(new News(i, "标题 " + i, "内容 " + i));
        }
        adapter.submitList(newsList);  // 提交数据，自动Diff + 动画（调了50次notifyItemInserted）
    }

    // 模拟刷新数据
    private void refreshNewsList() {
        List<News> newList = new ArrayList<>();
        for (int i = 0; i < 30; i++) {
            newList.add(new News(i, "新标题 " + i, "新内容 " + i));
        }
        adapter.submitList(newList);  // 自动比较差异，精准刷新 + 动画
    }
}
```

5）总结：

完整代码结构：

```
NewsDiffCallback.java  → 比较逻辑（id 是否相同、内容是否相同）
NewsAdapter.java       → 继承 ListAdapter，用 getItem() 取数据
MainActivity.java      → 调用 adapter.submitList(newList) 更新数据
```

关键点总结：

```
submitList(newList)：提交新数据，后台异步计算 Diff，自动刷新
areItemsTheSame：判断是否是同一个 item（比 id）
areContentsTheSame：判断内容是否变化（比字段）
getItem(position)：获取指定位置的数据
```

#### 多种 item 类型

思路：

1. 定义一个统一的数据模型，包含类型字段
2. 为每种类型创建单独的布局文件
3. 重写 `getItemViewType(position)` 返回类型
4. `onCreateViewHolder()` 根据类型加载不同布局
5. `onBindViewHolder()` 根据类型绑定不同数据

1）定义数据模型

用类型常量区分不同 item：

```java
public class SettingItem {
    public static final int TYPE_SWITCH = 0;
    public static final int TYPE_SEGMENT = 1;
    public static final int TYPE_BUTTON = 2;

    public int type;        // item类型
    public String title;    // 标题
    public String subtitle; // 副标题

    // Switch专用
    public boolean isChecked;

    // Segment专用
    public int selectedIndex;
    public String[] options;

    public SettingItem(int type, String title) {
        this.type = type;
        this.title = title;
    }
}
```

2）创建多个布局文件

item_switch.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp"
    android:gravity="center_vertical">

    <TextView
        android:id="@+id/titleTextView"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:textSize="16sp"/>

    <androidx.appcompat.widget.SwitchCompat
        android:id="@+id/switchView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>

</LinearLayout>
```

item_segment.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/titleTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        android:layout_marginBottom="8dp"/>

    <com.google.android.material.button.MaterialButtonToggleGroup
        android:id="@+id/segmentGroup"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:singleSelection="true">

        <com.google.android.material.button.MaterialButton
            android:id="@+id/option1"
            style="?attr/materialButtonOutlinedStyle"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"/>

        <com.google.android.material.button.MaterialButton
            android:id="@+id/option2"
            style="?attr/materialButtonOutlinedStyle"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"/>

        <com.google.android.material.button.MaterialButton
            android:id="@+id/option3"
            style="?attr/materialButtonOutlinedStyle"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"/>

    </com.google.android.material.button.MaterialButtonToggleGroup>

</LinearLayout>
```

item_button.xml：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp"
    android:gravity="center">

    <Button
        android:id="@+id/button"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</LinearLayout>
```

3）Adapter 实现多 ViewType

```java
public class SettingAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private List<SettingItem> itemList;

    public SettingAdapter(List<SettingItem> itemList) {
        this.itemList = itemList;
    }

    // 根据position返回对应的类型
    @Override
    public int getItemViewType(int position) {
        return itemList.get(position).type;
    }

    // 根据viewType创建不同的ViewHolder
    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        switch (viewType) {
            case SettingItem.TYPE_SWITCH:
                View switchView = inflater.inflate(R.layout.item_switch, parent, false);
                return new SwitchViewHolder(switchView);
            case SettingItem.TYPE_SEGMENT:
                View segmentView = inflater.inflate(R.layout.item_segment, parent, false);
                return new SegmentViewHolder(segmentView);
            case SettingItem.TYPE_BUTTON:
                View buttonView = inflater.inflate(R.layout.item_button, parent, false);
                return new ButtonViewHolder(buttonView);
            default:
                throw new IllegalArgumentException("未知的 viewType: " + viewType);
        }
    }

    // 根据ViewHolder类型绑定不同的数据
    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
        SettingItem item = itemList.get(position);
        switch (item.type) {
            case SettingItem.TYPE_SWITCH:
                ((SwitchViewHolder) holder).bind(item);
                break;
            case SettingItem.TYPE_SEGMENT:
                ((SegmentViewHolder) holder).bind(item);
                break;
            case SettingItem.TYPE_BUTTON:
                ((ButtonViewHolder) holder).bind(item);
                break;
        }
    }

    @Override
    public int getItemCount() {
        return itemList.size();
    }

    // ========== 三种 ViewHolder ==========

    static class SwitchViewHolder extends RecyclerView.ViewHolder {
        TextView titleTextView;
        SwitchCompat switchView;

        public SwitchViewHolder(@NonNull View itemView) {
            super(itemView);
            titleTextView = itemView.findViewById(R.id.titleTextView);
            switchView = itemView.findViewById(R.id.switchView);
        }

        public void bind(SettingItem item) {
            titleTextView.setText(item.title);
            switchView.setChecked(item.isChecked);
            switchView.setOnCheckedChangeListener((buttonView, isChecked) -> {
                item.isChecked = isChecked;
            });
        }
    }

    static class SegmentViewHolder extends RecyclerView.ViewHolder {
        TextView titleTextView;
        MaterialButtonToggleGroup segmentGroup;
        MaterialButton option1, option2, option3;

        public SegmentViewHolder(@NonNull View itemView) {
            super(itemView);
            titleTextView = itemView.findViewById(R.id.titleTextView);
            segmentGroup = itemView.findViewById(R.id.segmentGroup);
            option1 = itemView.findViewById(R.id.option1);
            option2 = itemView.findViewById(R.id.option2);
            option3 = itemView.findViewById(R.id.option3);
        }

        public void bind(SettingItem item) {
            titleTextView.setText(item.title);
            option1.setText(item.options[0]);
            option2.setText(item.options[1]);
            option3.setText(item.options[2]);
            segmentGroup.check(getButtonId(item.selectedIndex));
        }

        private int getButtonId(int index) {
            switch (index) {
                case 0: return option1.getId();
                case 1: return option2.getId();
                case 2: return option3.getId();
                default: return option1.getId();
            }
        }
    }

    static class ButtonViewHolder extends RecyclerView.ViewHolder {
        Button button;

        public ButtonViewHolder(@NonNull View itemView) {
            super(itemView);
            button = itemView.findViewById(R.id.button);
        }

        public void bind(SettingItem item) {
            button.setText(item.title);
            button.setOnClickListener(v -> {
                // 处理点击
            });
        }
    }
}
```

4）Activity 中使用

```java
public class MainActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private SettingAdapter adapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        recyclerView = findViewById(R.id.recyclerView);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        List<SettingItem> itemList = createSettingItems();
        adapter = new SettingAdapter(itemList);
        recyclerView.setAdapter(adapter);
    }

    private List<SettingItem> createSettingItems() {
        List<SettingItem> list = new ArrayList<>();

        // Switch 类型
        SettingItem wifi = new SettingItem(SettingItem.TYPE_SWITCH, "Wi-Fi");
        wifi.isChecked = true;
        list.add(wifi);

        SettingItem bluetooth = new SettingItem(SettingItem.TYPE_SWITCH, "蓝牙");
        bluetooth.isChecked = false;
        list.add(bluetooth);

        // Segment 类型
        SettingItem theme = new SettingItem(SettingItem.TYPE_SEGMENT, "主题模式");
        theme.options = new String[]{"浅色", "深色", "跟随系统"};
        theme.selectedIndex = 0;
        list.add(theme);

        // Button 类型
        SettingItem logout = new SettingItem(SettingItem.TYPE_BUTTON, "退出登录");
        list.add(logout);

        return list;
    }
}
```

6）总结

完整代码结构：

```
SettingItem.java           → 统一数据模型（含 type 字段）
SettingAdapter.java        → Adapter + 3 种 ViewHolder
item_switch.xml            → Switch 类型布局
item_segment.xml           → Segment 类型布局
item_button.xml            → Button 类型布局
activity_main.xml          → 主布局
```

关键点总结：

```
getItemViewType(position)：返回该位置的 item 类型
onCreateViewHolder(parent, viewType)：根据 viewType 加载不同布局
onBindViewHolder(holder, position)：根据 holder 类型绑定不同数据
```











































