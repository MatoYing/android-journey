#### 基本用法

1）引入依赖

```groovy
implementation 'androidx.recyclerview:recyclerview:1.1.0'
```

2）添加 RecyclerView 到布局文件

```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

3）添加一个 `item_list.xml` 的布局文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/textView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="29dp"
        android:layout_marginTop="33dp"
        android:text="标题"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/textView2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="4dp"
        android:layout_marginTop="18dp"
        android:text="内容"
        app:layout_constraintStart_toStartOf="@+id/textView"
        app:layout_constraintTop_toBottomOf="@+id/textView" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

4）自定义 Adapter（把数据绑定到每个 item 上）和 ViewHolder（和 item 视图绑定，复用时直接使用，不重新创造视图）

```java
public class MainActivity extends AppCompatActivity {
    RecyclerView mRecyclerView;
    MyAdapter mMyAdapter ;
    List<News> mNewsList = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      
        mRecyclerView = findViewById(R.id.recyclerview);
        // 构造一些数据
        for (int i = 0; i < 50; i++) {
            News news = new News();
            news.title = "标题" + i;
            news.content = "内容" + i;
            mNewsList.add(news);
        }
        mMyAdapter = new MyAdapter();
        mRecyclerView.setAdapter(mMyAdapter);
        LinearLayoutManager layoutManager = new LinearLayoutManager(MainActivity.this);
        mRecyclerView.setLayoutManager(layoutManager);
    }

    class MyAdapter extends RecyclerView.Adapter<MyViewHoder> {

        @NonNull
        @Override
        public MyViewHoder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
            View view = View.inflate(MainActivity.this, R.layout.item_list, null);
            MyViewHoder myViewHoder = new MyViewHoder(view);
            return myViewHoder;
        }

        @Override
        public void onBindViewHolder(@NonNull MyViewHoder holder, int position) {
            News news = mNewsList.get(position);
            holder.mTitleTv.setText(news.title);
            holder.mTitleContent.setText(news.content);
        }

        @Override
        public int getItemCount() {
            return mNewsList.size();
        }
    }

    class MyViewHoder extends RecyclerView.ViewHolder {
        TextView mTitleTv;
        TextView mTitleContent;

        public MyViewHoder(@NonNull View itemView) {
            super(itemView);
            mTitleTv = itemView.findViewById(R.id.textView);
            mTitleContent = itemView.findViewById(R.id.textView2);
        }
    }
}
```

其他：

1）布局管理器

出了上面提到的 `LinearLayoutManager layoutManager = new LinearLayoutManager(MainActivity.this);`，RecyclerView 提供了三种布局管理器：

```java
// 垂直列表
recyclerView.setLayoutManager(new LinearLayoutManager(this));
// 网格
recyclerView.setLayoutManager(new GridLayoutManager(this, 2));
// 瀑布流
recyclerView.setLayoutManager(new StaggeredGridLayoutManager(2, StaggeredGridLayoutManager.VERTICAL));
```

2）点击事件

```java
holder.itemView.setOnClickListener(v -> {
    int pos = holder.getAdapterPosition();
    // 响应点击
});
```

3）Item 分隔线

```java
recyclerView.addItemDecoration(new DividerItemDecoration(context, DividerItemDecoration.VERTICAL));
```

4）数据更新

```java
// 整体刷新
dataList.clear();
dataList.addAll(newData);
adapter.notifyDataSetChanged();


// 局部刷新
// 插入第 2 项
dataList.add(1, "新数据");
adapter.notifyItemInserted(1);

// 删除第 3 项
dataList.remove(2);
adapter.notifyItemRemoved(2);

// 更新第 1 项
dataList.set(0, "更新后的内容");
adapter.notifyItemChanged(0);


// 范围性刷新
adapter.notifyItemRangeInserted(start, count);
adapter.notifyItemRangeRemoved(start, count);
adapter.notifyItemRangeChanged(start, count);
```

5）Item 动画

```java
DefaultItemAnimator itemAnimator = new DefaultItemAnimator();
defaultItemAnimator.setAddDuration(1000);
defaultItemAnimator.setRemoveDuration(1000);
mRecyclerView.setItemAnimator(itemAnimator);
```







