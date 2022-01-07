# BindingAdapter

之前在 LiveData 的代码中，为了测试 Activity 位于后台时，不会通知 Observer LiveData 有更新，我在 ViewModel 中添加了一个自增计数的后台线程。

```java
public class MainViewModel extends ViewModel {
    ...
    private volatile boolean runThread = false;

    private final Runnable task = () -> {
        runThread = true;
        while (runThread) {
            getModel().increment();
            liveData.postValue(getModel());
            SystemClock.sleep(1000);
        }
    };
}
```

为了能在 UI 中启用这个线程，需要扩展操作逻辑：
- 默认为手动模式，点 button 一次增加一个数
- 长按 button 进入自动模式，点一次启用后台线程，再点一次关闭后台线程。

将 ViewModel 添加如下代码：

```java
public class MainViewModel extends ViewModel {
    ...
    // 响应长按操作
    public boolean toggleIncrementMode() {
        switch (Objects.requireNonNull(incrementMode.getValue())) {
            case MANUAL_MODE:
                prepareAutoIncrementMode();
                break;
            case AUTO_TASK_STOPED:
                toggleManualMode();
                break;
            case AUTO_TASK_STARTED:
                exitAutoIncrementMode();
                break;
        }
        // 长按操作需要返回 true
        return true;
    }

    // 切换手动模式
    private void toggleManualMode() {
        incrementMode.setValue(IncrementMode.MANUAL_MODE);
    }

    // 切换自动模式
    private void prepareAutoIncrementMode() {
        incrementMode.setValue(IncrementMode.AUTO_TASK_STOPED);
    }
    
    // 退出自动模式，并关闭后台线程
    private void exitAutoIncrementMode() {
        incrementMode.setValue(IncrementMode.MANUAL_MODE);
        runThread = false;
    }

    public MutableLiveData<IncrementMode> getIncrementMode() {
        return incrementMode;
    }
    
    // 记录当前模式
    private final MutableLiveData<IncrementMode> incrementMode = new MutableLiveData<>(IncrementMode.MANUAL_MODE);
    
    // 响应点击操作
    public void handleIncrement() {
        switch (Objects.requireNonNull(incrementMode.getValue())) {
            case MANUAL_MODE:
                getModel().increment();
                break;
            case AUTO_TASK_STARTED:
                stopAutoIncrement();
                break;
            case AUTO_TASK_STOPED:
                startAutoIncrement();
                break;
        }
        mainModelLiveData.setValue(getModel());
    }

    // 开启后台线程
    private void startAutoIncrement() {
        incrementMode.setValue(IncrementMode.AUTO_TASK_STARTED);
        new Thread(task).start();
    }
    
    // 关闭后台线程
    private void stopAutoIncrement() {
        incrementMode.setValue(IncrementMode.AUTO_TASK_STOPED);
        runThread = false;
    }
    ...
}
```

UIController 中则需要为 incrementMode 设置 Observer ，并根据 incrementMode 枚举值设置 button icon：
```java
public class MainActivity extends AppCompatActivity {

    MainViewModel viewModel;
    ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        viewModel.getIncrementMode().observe(this, new Observer<IncrementMode>() {
            @Override
            public void onChanged(IncrementMode incrementMode) {
                fab.setImageDrawable(getDrawerMainFab(incrementMode, getApplicationContext()));
            }
        });
        ...
        fab.setOnClickListener((view) ->
                viewModel.handleIncrement()
        );

        fab.setOnLongClickListener((view) ->
                viewModel.toggleIncrementMode()
        );
    }

    private Drawable getDrawerMainFab(IncrementMode incrementMode, Context context) {
        switch (incrementMode) {
            case AUTO_TASK_STARTED:
                return ContextCompat.getDrawable(context, R.drawable.ic_pause);
            case AUTO_TASK_STOPED:
                return ContextCompat.getDrawable(context, R.drawable.ic_play);
            default:
                return ContextCompat.getDrawable(context, R.drawable.ic_add);
        }
    }
}
```

现在引入了 DataBinding , UIController 中设置 Observer 的代码不需要了，UI 交互代码也转移到 Layout 中：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name = "data"
            type="com.codingnight.android.databinding.MainViewModel"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">

        ...

        <com.google.android.material.floatingactionbutton.FloatingActionButton
            ...
            android:onClick="@{()->data.toggleIncrement()}"
            android:onLongClick="@{()->data.toggleIncrementMode()}"
            ... />

            ...

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```

那么，设置 button icon 呢？难道还要保留设置 Observer ？这也太不 Databinding 了。

好在 DataBinding 库提供了 BindingAdapter ，在 BindingAdpater 中实现上述逻辑，然后在 Layout 中使用 BindingAdapter 。

BindingAdapter 本质上就是新建 View 属性或者覆盖现有的 View 属性。

例如这里使用 `@BindingAdapter` 注解为 FloatingActionButton 新建了 app:mainFabIcon 属性：
- mainFabIcon 方法的第一个参数指定了该属性能应用的 View 类
- mainFabIcon 方法同时可以接受 IncrementMode 枚举值，根据枚举值来为 FloatingActionButton 设置 icon 。

```java
public class BindingAdapters {

    @BindingAdapter("app:mainFabIcon")
    public static void mainFabIcon(FloatingActionButton fab, IncrementMode incrementMode) {
        fab.setImageDrawable(getDrawerMainFab(incrementMode, button.getContext()));
    }

    private static Drawable getDrawerMainFab(IncrementMode incrementMode, Context context) {
        switch (incrementMode) {
            case AUTO_TASK_STARTED:
                return ContextCompat.getDrawable(context, R.drawable.ic_pause);
            case AUTO_TASK_STOPED:
                return ContextCompat.getDrawable(context, R.drawable.ic_play);
            default:
                return ContextCompat.getDrawable(context, R.drawable.ic_add);
        }
    }
}
```

然后就可以在 Layout 为 FloatingActionButton 设置 app:mainFabIcon 属性：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name = "data"
            type="com.codingnight.android.databinding.MainViewModel"/>
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">

        ...

        <com.google.android.material.floatingactionbutton.FloatingActionButton
            ...
            app:mainFabIcon="@{data.incrementMode}"
            ... />

            ...

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```

这里使用绑定表达式将 incrementMode 传参给 BindingAdapter 。

到这里，完成了使用 BindingAdapter 将设置 button icon 逻辑从 UIController 中分离。