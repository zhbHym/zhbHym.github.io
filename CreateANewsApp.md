# 新闻APP

  第一次写代码笔记，难免会出现不知道写什么东西。废话不说了，开始吧。
参考自文档： http://www.jianshu.com/p/c2e79cf83c32
数据来自[天行数据](http://www.tianapi.com/ "天行数据")，只需要注册就行。 

## 使用框架：
rxjava和retrofit以及一个开源扩展的RecyclerView和注解框架butterknife。
在工程的build.gradle的dependencies中添加

    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    

在项目build.gradle中需要添加如下代码：
~~~
apply plugin: 'com.neenbedankt.android-apt'
......
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:24.2.1'
    compile 'com.android.support:design:24.2.1'
    testCompile 'junit:junit:4.12'
    //	依赖注解
    //	依赖添加
    compile 'com.jakewharton:butterknife:8.4.0'
    apt 'com.jakewharton:butterknife-compiler:8.4.0'
    compile 'com.google.code.gson:gson:2.7'
    //	开源的高级 recyclerview
    compile 'com.jude:easyrecyclerview:4.2.3'
    compile 'com.android.support:recyclerview-v7:24.2.0'
    //	rxjava
    compile 'com.squareup.retrofit2:retrofit-converters:2.1.0'
    compile 'com.squareup.retrofit2:converter-gson:2.1.0'
    compile 'io.reactivex:rxandroid:1.2.1'
    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
    compile 'com.squareup.retrofit2:retrofit:2.1.0'
    compile 'com.squareup.retrofit2:converter-scalars:2.1.0'
    compile 'com.github.bumptech.glide:glide:3.7.0'
}
~~~

千万别忘了添加权限：
~~~
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS" />
~~~
## 开始制作

我采用了fragment加载界面的方法，所以新建一个fragment，通过动态添加的方式添加fragment,Activity添加Fagment的方法如下：
~~~
 // 1、获取 FragmentManager
 FragmentManager fragmentManager = getSupportFragmentManager();
 // 2、声明并获得 FragmentTransaction
 FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
 // 3、 声明自定义的 Fragment
 NewsFragment newsFragment = new NewsFragment();
 // 4、 调用 FragmentTransaction 的replace方法，
 fragmentTransaction.replace(R.id.fragment_content,newsFragment);
 // 5、 最后提交FragmentTransaction
 fragmentTransaction.commit();
~~~

在NewsFragment中，复写onCreateView方法初始化页面，加载控件的小技巧：

    @BindView(R.id.recyclerview)
    EasyRecyclerView recyclerView;

在该方法中，可以添加下拉刷新、加载更多事件以及点击详情事件。
下拉刷新事件复写recyclerView的setRefreshListener即可为：
~~~
recyclerView.setRefreshListener(new SwipeRefreshLayout.OnRefreshListener() {
            @Override
            public void onRefresh() {
                recyclerView.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        adapter.clear();//清楚数据
                        page = 0; //全局变量，加载的页数
                        getData(); //加载数据的方法
                    }
                }, 1000);
            }
        });
~~~
加载更多需要复写adapter的setMore方法，具体代码如下：
~~~
adapter.setMore()
{
@Override
            public void onMoreShow() {
                getData();//加载数据的方法
            }
            @Override
            public void onMoreClick() {
            }
}
~~~
加载详情的方法，复写setOnItemClickListener即可
~~~
//点击事件
adapter.setOnItemClickListener(new RecyclerArrayAdapter.OnItemClickListener() {
    @Override
    public void onItemClick(int position) {  //position中携带item 的位置
        ArrayList<String> data = new ArrayList<String>();
        data.add(adapter.getAllData().get(position).getPicUrl());
        data.add(adapter.getAllData().get(position).getUrl());
        Intent intent = new Intent(getActivity(), NewsDetailsActivity.class);
        //用Bundle携带数据
        Bundle bundle = new Bundle();
        bundle.putStringArrayList("data", data);
        intent.putExtras(bundle);
        startActivity(intent);
        }
});
~~~

##### 下面就是最最重要的了，在onViewCreated中加载数据，
其实该方法很简单，调用getData()方法即可。
最重要的代码，数据获取函数getData()方法：
步骤如下：
	1、使用retrofit 发起网络请求
	2、数据通过rxjava提交先在io线程里，返回到主线程
	3、中间设置map转换，把得到的Gson类转化为所需的News类（可以省略这一步）
	4、subscribe的onNext里处理返回的最终数据。
具体代码为：
~~~
private void getData() {
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://api.tianapi.com/")
                //String
                .addConverterFactory(ScalarsConverterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                //添加 json 转换器
                //    compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                //添加 RxJava 适配器
                .build();

        ApiService apiManager = retrofit.create(ApiService.class);
        //这里采用的是Java的动态代理模式
        apiManager.getNewsData("0271191a3d0bcd8483debff0c759f20a", 10, page)
                .subscribeOn(Schedulers.io())
                .map(new Func1<NewsGson, List<News>>() {
                    @Override
                    public List<News> call(NewsGson newsgson) { //
                        List<News> newsList = new ArrayList<News>();
                        for (NewsGson.NewslistBean newslistBean : newsgson.getNewslist()) {
                            News new1 = new News();
                            new1.setTitle(newslistBean.getTitle());
                            new1.setCtime(newslistBean.getCtime());
                            new1.setDescription(newslistBean.getDescription());
                            new1.setPicUrl(newslistBean.getPicUrl());
                            new1.setUrl(newslistBean.getUrl());
                            newsList.add(new1);
                        }
                        Log.d("news",newsList.toString());
                        return newsList; // 返回类型
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<List<News>>() {
                    @Override
                    public void onNext(List<News> newsList) {
                        adapter.addAll(newsList);
                    }
                    @Override
                    public void onCompleted() {
                    }
                    @Override
                    public void onError(Throwable e) {
                        Toast.makeText(getContext(),
                                "网络连接失败", Toast.LENGTH_LONG).show();
                    }
                });
        page = page + 1;
    }
}
~~~

##### 自定义Service，采用Java的动态代理模式
~~~
public interface ApiService {

    //返回的是Gson数据而且设置为"被观察者"
    @GET("social/")
    Observable <NewsGson> getNewsData(@Query("key")String key,@Query("num") int num,@Query("page") int page);

}
~~~

#####通过adapter加载数据
~~~
public class NewsAdapter extends RecyclerArrayAdapter<News> {

    public NewsAdapter(Context context)
    {
        super(context);
    }

    @Override
    public BaseViewHolder OnCreateViewHolder(ViewGroup parent, int viewType) {
        return new NewsViewHolder(parent);//调用NewsViewHolder
    }
}
~~~
通过NewsViewHolder显示数据
~~~
public class NewsViewHolder extends BaseViewHolder<News>{

    private TextView mTv_name;
    private TextView mTv_sign;
    private ImageView mImg_face;

    public NewsViewHolder(ViewGroup parent) {
        super(parent, R.layout.news_recycler_item);
        //	加载控件
        mImg_face = $(R.id.person_face);
        mTv_name = $(R.id.person_name);
        mTv_sign = $(R.id.person_sign);
    }

    @Override
    public void setData(News data) {
        super.setData(data);
        //
        mTv_name.setText(data.getTitle());
        mTv_sign.setText(data.getDescription());
        Glide.with(getContext())   //网络下载并加载图片
                .load(data.getPicUrl())
                .placeholder(R.mipmap.ic_launcher)
                .centerCrop()
                .into(mImg_face);
    }
}
~~~
###### 到这里，新闻加载就基本完成了，下面给出加载图片的方式，使图片能够不变形。

### 加载图片

这里只需要定义一个fragment，当点击了菜单栏中的 每日美图 时动态的加载界面。

  该Fragment中，和新闻界面一样，在onCreateView中初始化控件，添加下拉刷新、加载更多、点击等事件，还有重要的addData()方法。下面直接看addData方法,其实和加载新闻的差不多，也是几个步骤。
~~~
private void addData() {
        Retrofit retrofit = new Retrofit.Builder() // 发起网络请求
                .baseUrl("http://api.tianapi.com/")
                .addConverterFactory(GsonConverterFactory.create()) 
                // 添加Gson转换器
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                //  添加Rxjava适配器
                .build();

		// 采用Java动态代理模式
        ApiService apiManager = retrofit.create(ApiService.class);
        apiManager.getPictureData("8fdda4c5c6876bf06524cecaef410eda",10,page)
                .subscribeOn(Schedulers.io()) //提交数据到io线程中
                .map(new Func1<MeiNvGson, List<MeiNv>>() {
                    @Override
                    public List<MeiNv> call(MeiNvGson meiNvGson) {

                        List<MeiNv> meiNvList = new ArrayList<MeiNv>();
                        for (MeiNvGson.NewslistBean newslistBean: meiNvGson.getNewslist())
                        {
                            MeiNv m1 = new MeiNv();
                            m1.setTitle(newslistBean.getTitle());
                            m1.setCtime(newslistBean.getCtime());
                            m1.setDescription(newslistBean.getDescription());
                            m1.setPicUrl(newslistBean.getPicUrl());
                            m1.setUrl(newslistBean.getUrl());
                            meiNvList.add(m1);
                        }
                        return meiNvList;
                    }
                })
                .observeOn(AndroidSchedulers.mainThread()) // 返回主线程
                .subscribe(new Subscriber<List<MeiNv>>() {

                    @Override
                    public void onNext(List<MeiNv> meiNvList) {

                        adapter.addAll(meiNvList);
                    }
                    @Override
                    public void onCompleted() {
                    }

                    @Override
                    public void onError(Throwable e) {

                        Toast.makeText(getContext(),"连接网络失败",Toast.LENGTH_LONG).show();
                    }
                });

        page = page + 1;
    }
~~~
适配器的代码为：
~~~
public class ImageAdapter extends RecyclerArrayAdapter <MeiNv> {

    public ImageAdapter(Context context)
    {
        super(context);
    }

    @Override
    public BaseViewHolder OnCreateViewHolder(ViewGroup parent, int viewType) {
        return new ImageViewHolder(parent);
    }
}
~~~

~~~
public class ImageViewHolder extends BaseViewHolder<MeiNv>{

    ImageView imgPicture;
    public ImageViewHolder(ViewGroup parent){
        super(new ImageView(parent.getContext()));

        imgPicture = (ImageView) itemView;
        imgPicture.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT));
        imgPicture.setScaleType(ImageView.ScaleType.CENTER_CROP);
    }

    @Override
    public void setData(MeiNv data) {

        final DisplayMetrics dm = getContext().getResources().getDisplayMetrics();
        Glide.with(getContext())
                .load(data.getPicUrl())
                .asBitmap()
                .into(new SimpleTarget<Bitmap>(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL) {
                    @Override
                    public void onResourceReady(Bitmap bitmap, GlideAnimation<? super Bitmap> glideAnimation) {
                        int width = dm.widthPixels / 2 - 10;//宽度为屏幕宽度一半
                        int heigh = bitmap.getHeight() * width / bitmap.getWidth(); //计算View的高度
                        //获取bitmap信息，可赋值给外部变量操作，也可在此时行操作。
                        ViewGroup.LayoutParams params = imgPicture.getLayoutParams();
                        params.height = heigh;
                        params.width = width;
                        imgPicture.setLayoutParams(params);
                        imgPicture.setScaleType(ImageView.ScaleType.FIT_XY);
                        imgPicture.setImageBitmap(bitmap);
                    }
                });
    }
}
~~~

到这里基本上就完成了，下面记录两个用到的小知识。
***

##### 1、保存图片
可以给ImageView添加长按监听事件（setOnLongClickListener），在里面调用保存图片函数，保存图片到本地的函数如下：
~~~
public static void saveImage(ImageView shot, Context context){

		// 获取外部存储目录
        File externalStorageDirectory = Environment.getExternalStorageDirectory();
        // 新建目录
        File directory = new File(externalStorageDirectory, "NewsPicture");
        if (! directory.exists())
        {
      		// File.mkdirs()方法可以建好一个完整的路径
            // File.mkdir()  只能在已经存在的目录中创建创建文件夹。 
            directory.mkdir(); //创建文件
        }
        Bitmap drawingCache = shot.getDrawingCache();

        File file = new File(directory, new Date().getTime() + ".jpg");
        try {
            FileOutputStream fos = new FileOutputStream(file);
            // 保存图片
            drawingCache.compress(Bitmap.CompressFormat.JPEG,100,fos);
            // 发送广播
            Intent intent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
            Uri uri = Uri.fromFile(file);
            intent.setData(uri);
            context.sendBroadcast(intent);
            Toast.makeText(context,"保存成功",Toast.LENGTH_LONG).show();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
            Toast.makeText(context,"保存出错",Toast.LENGTH_LONG).show();
        }
    }
~~~

##### 2、按照图片比例设置图片大小
在加载图片过程中，相信大家都遇到这样的问题：由于没有按照图片的比例加载图片，加载后会出现图片失真的现象。下面就教大家如何加载的图片不会失真，还是在代码中看吧。
~~~
// 获取屏幕大小
final DisplayMetrics dm = getResources().getDisplayMetrics();
// 设置宽度为屏幕宽度
int width = dm.widthPixels;
// 设置高度为 图片高度 * 屏幕宽度 / 图片宽度
int height = bitmap.getHeight()*width/bitmap.getWidth();
// 获取ImageView尺寸
ViewGroup.LayoutParams params = imgPicture.getLayoutParams();
params.width = width;
params.height = height;
imgPicture.setLayoutParams(params);
// 设置图片填充满 ImageView
imgPicture.setScaleType(ImageView.ScaleType.FIT_XY);
imgPicture.setImageBitmap(resource);
~~~











