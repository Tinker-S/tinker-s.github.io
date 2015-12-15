title: 开发Moto360国行版手表应用
---
Moto360二代国行版手表已经发布很长时间了，然而很多开发者却不知道怎么适配自己的应用使其正常运行。本文会一步步展现如何将android sdk sample中的DataLayer工程适配Moto360国行版。这个工程本来是运行在原生android wear系统上。

### 下载国行版专用SDK

Moto国行版应用手机端必须使用国行版本专门定制的SDK，使用和android wear原生一样的play-service包是无法和手表进行通信的，手表端仍然可以使用play-services-wearable包。

那么如何下载SDK呢，[点击下载](https://github.com/ticwear/sdk/raw/master/android-wear-lib/wearable-api-client-repository.zip)

下载完成解压之后我们顺着路径点开可以看到一个文件`play-services-wearable-standalone-7.5.0.aar`

### 准备DataLayer工程

Android SDK的sample目录下有一个DataLayer的工程，是演示手机和手表如何使用gms进行数据通信的，路径在 `$ANDROID_HOME/samples/android-22/wearable/DataLayer`。我们将其拷贝出来，放在自己的工作目录下。

### 替换手机端SDK引用

手机端需要将之前引用的`com.google.android.gms:play-services-wearable`替换为我们下载的SDK。

在Application模块下新建libs目录，并将之前提取出来的aar文件放置到libs目录下；

修改Application模块的build.gradle，关键代码如下：

``` bash
repositories {
    flatDir {
        dirs 'libs'
    }
}

dependencies {
    compile(name:'play-services-wearable-standalone-7.5.0', ext:'aar')
}
```

### 修改调用代码

替换了手机端SDK引用之后我们编译代码，发现编译不通过，查看之下原来是Application模块里面MainActivity的一段代码的问题：

``` java
@Override //DataListener
public void onDataChanged(DataEventBuffer dataEvents) {
    LOGD(TAG, "onDataChanged: " + dataEvents);
    final List<DataEvent> events = FreezableUtils.freezeIterable(dataEvents);
    dataEvents.close();
    runOnUiThread(new Runnable() {
        @Override
        public void run() {
            for (DataEvent event : events) {
                if (event.getType() == DataEvent.TYPE_CHANGED) {
                    mDataItemListAdapter.add(
                            new Event("DataItem Changed", event.getDataItem().toString()));
                } else if (event.getType() == DataEvent.TYPE_DELETED) {
                    mDataItemListAdapter.add(
                            new Event("DataItem Deleted", event.getDataItem().toString()));
                }
            }
        }
    });
}
```

错误原因是国行版SDK并没有提供FreezableUtils这个类，没关系，我们自己来实现这一块，修改之后的代码：

``` java
@Override //DataListener
public void onDataChanged(final DataEventBuffer dataEvents) {
    LOGD(TAG, "onDataChanged: " + dataEvents);
    final ArrayList<DataEvent> events = new ArrayList<DataEvent>();
    for (DataEvent event : dataEvents) {
        events.add(event.freeze());
    }
    dataEvents.release();
    runOnUiThread(new Runnable() {
        @Override
        public void run() {
            for (DataEvent event : events) {
                if (event.getType() == DataEvent.TYPE_CHANGED) {
                    mDataItemListAdapter.add(
                            new Event("DataItem Changed", event.getDataItem().toString()));
                } else if (event.getType() == DataEvent.TYPE_DELETED) {
                    mDataItemListAdapter.add(
                            new Event("DataItem Deleted", event.getDataItem().toString()));
                }
            }
        }
    });
}
```

编译，安装，运行，发现一切都很完美。

### 总结

至此我们的适配工作就全部完成了。细心的朋友会发现，Android Wear原生app适配国行版其实很简单，就是需要将手机端引用的play-service包替换成国行版本专用的SDK即可，无需做其他额外的修改。

### Demo链接

具体的代码参见 [Demo](https://github.com/Tinker-S/DataLayerForMotoChina)，有任何问题可发邮件至`sunting.bcwl@gmail.com`。