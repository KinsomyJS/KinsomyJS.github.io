---
layout:     post                    # 使用的布局（不需要改）
title:   Glide替换Picasso          # 标题 
subtitle:   Migrate from Picasso to Glide #副标题
date:       2019-03-15            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 背景
之前的项目中有一些较老的模块使用了Picasso。

因为前期项目没有做组件化，缺少底层基础设施库，并且Picasso没有支持Gif加载，Glide又逐渐成为谷歌官方推荐的图片库，所以后面开发的新模块和APP都使用到了Glide，项目中存在两套图片加载库的情况。

因此需要用Glide替换Picasso库，为组件化做准备。

## 2 对比

Glide 和 Picasso在API的调用上 非常相似，且都支持图片的内存缓存，都是非常优秀的图片加载框架，可以说Glide是Picasso的升级，在性能上有所提升。

### 2.1 缓存

首先Picasso是**2级缓存**，它支持内存缓存而不支持磁盘缓存；而Glide是**3级缓存**，也就是说依次按照**内存 > 磁盘 >网络**的优先级来加载图片。

再者，二者图片缓存的策略不同。将同一张网络图片加载到相同大小的ImageView中，Glide 加载的图片质量是不如Picasso的，原因是：Glide 加载图片默认的 Bitmap 格式是 RGB-565，一个像素点占16位 ，而 Picasso默认的格式是ARGB-8888 ，一个像素点占32位，所以Glide的内存开销要小一半。当然 Glide也可以通过GlideModule 将 Bitmap 格式转换到 ARGB-8888。

> Android 8.0之前 Bitmap存放在Java Heap中，如果内存占用过大会出现OOM，8.0之后存放到Native Heap中。

还有一点， Picasso 是加载了全尺寸的图片到内存，下次在任何ImageView中加载图片的时候，全尺寸的图片将从缓存中取出，重新调整大小，然后缓存。而 Glide 是按 ImageView 的大小来缓存的，它会为每种大小的ImageView缓存一次。尽管一张图片已经缓存了一次，但是假如你要在另外一个地方再次以不同尺寸显示，需要重新下载，调整成新尺寸的大小，然后将这个尺寸的也缓存起来。具体说来就是：假如在第一个页面有一个200x200的ImageView，在第二个页面有一个100x100的ImageView，这两个ImageView本来是要显示同一张图片，却需要下载两次。

> 结论：Glide的这种方式优点是加载显示非常快，但同时也需要更大的空间来缓存。

### 2.2 生命周期
Glide 相比 Picasso 的一大优势是它可以和 Activity 以及 Fragment 的生命周期相互协作，我们在调用 Glide.with() 函数时可以将 Activity 或者 Fragment 的实例传进去，这样 Glide 就会自动将图片加载等操作和组件的生命周期关联起来。

### 2.3 额外功能
* Glide能够从视频加载帧作为视频缩略图

* Glide可以支持加载GIF图

## 3 实践

Glide和Picasso的Api很相似，所以正常的加载图片代码只需要对照Glide的文档替换即可。

项目里有一个将pdf文件转换成图片用Picasso加载的逻辑，Picasso用到了RequestHandler拦截pdf的url加载，然后用PdfRender转化成bitmap返回给Picasso加载并缓存，Glide有类似的功能，叫`ModelLoader`，具体的可以参考官方文档[编写定制的 ModelLoader ](https://muyangmin.github.io/glide-docs-cn/tut/custom-modelloader.html)。


先看一下Picasso的实现，用`addRequestHandler`添加拦截器，对load的url做识别，判断是pdf的scheme才处理请求，然后重写`load`方法，将pdf绘制到bitmap对象，然后将Bitmap返回给Picasso用于渲染到view上，同时还可以将bitmap缓存起来。

`Picasso`
```java 
picasso = new Picasso.Builder(mContext).addRequestHandler(new RequestHandler() {
            @Override
            public boolean canHandleRequest(Request data) {
                if (data.uri.getScheme().equals("pdf")) return true;
                return false;
            }

            @Nullable
            @Override
            public Result load(Request request, int networkPolicy) throws IOException {
                synchronized (picasso) {
                    int position = Integer.parseInt(request.uri.getQuery());
                    currentPage = mRenderer.openPage(position);
                    int pageWidth = currentPage.getWidth();
                    int pageHeight = currentPage.getHeight();
                    mPageRatio = (float)pageWidth/pageHeight;
                    Point point = getDispaly(mContext);
                    int dstwidth = point.x;
                    int dstheight = (int)(dstwidth/mPageRatio);

                    Bitmap bitmap = Bitmap.createBitmap(dstwidth, dstheight, Bitmap.Config.ARGB_8888);
                    drawBackground(bitmap);
                    currentPage.render(bitmap, null, null, PdfRenderer.Page.RENDER_MODE_FOR_DISPLAY);
                    currentPage.close();
                    currentPage = null;
                    return new Result(bitmap, Picasso.LoadedFrom.MEMORY);
                }
            }
        }).build();
```


`Glide`

```java 
@GlideModule
public class LibGlideModule extends LibraryGlideModule {
    @Override
    public void registerComponents(@NonNull Context context, @NonNull Glide glide, @NonNull Registry registry) {
        registry.prepend(PdfModel.class, Bitmap.class, new PdfGlideModuleFactory());
    }
}

public class PdfGlideModuleFactory implements ModelLoaderFactory<PdfModel, Bitmap> {
    @NonNull
    @Override
    public ModelLoader<PdfModel, Bitmap> build(@NonNull MultiModelLoaderFactory multiFactory) {
        return new PdfModelLoader();
    }

    @Override
    public void teardown() {

    }
}

public class PdfModelLoader implements ModelLoader<PdfModel, Bitmap> {

    @Nullable
    @Override
    public LoadData<Bitmap> buildLoadData(@NonNull PdfModel model, int width, int height, @NonNull Options options) {
        int position = Integer.parseInt(URI.create(model.uri).getQuery());
        return new LoadData<>(new ObjectKey(model.pdfPath + "/" + position), new PdfDataFetcher(model));
    }

    @Override
    public boolean handles(@NonNull PdfModel model) {
        if (model.uri.startsWith("pdf")) {
            return true;
        }
        return false;
    }
}


public class PdfModel {
    public String pdfPath;
    public Context mContext;
    public PdfRenderer mRenderer;
    public String uri;
    public PdfCallback mCallback;

    public PdfModel(Context mContext, PdfRenderer mRenderer, String uri, String pdfPath,PdfCallback pdfCallback) {
        this.mContext = mContext;
        this.mRenderer = mRenderer;
        this.uri = uri;
        this.pdfPath = pdfPath;
        this.mCallback = pdfCallback;
    }
}

interface PdfCallback {
    void loadData(PdfRenderer.Page page);
}

public class PdfDataFetcher implements DataFetcher<Bitmap> {
    PdfModel mPdfModel;

    public static final Object lock = new Object();

    PdfRenderer.Page mCurrentPage;

    public PdfDataFetcher(PdfModel pdfModel) {
        this.mPdfModel = pdfModel;
    }

    @Override
    public void loadData(@NonNull Priority priority, @NonNull DataCallback<? super Bitmap> callback) {
        // 要加锁顺序执行图片加载，不然会出现列表里随机图片item加载不出来
        synchronized (lock) {
            int position = Integer.parseInt(URI.create(mPdfModel.uri).getQuery());
            mCurrentPage = mPdfModel.mRenderer.openPage(position);
            if (mPdfModel.mCallback != null) {
                mPdfModel.mCallback.loadData(mCurrentPage);
            }
            int pageWidth = mCurrentPage.getWidth();
            int pageHeight = mCurrentPage.getHeight();
            float mPageRatio = (float) pageWidth / pageHeight;
            Point point = getDispaly(mPdfModel.mContext);
            int dstwidth = point.x;
            int dstheight = (int) (dstwidth / mPageRatio);

            Bitmap bitmap = Bitmap.createBitmap(dstwidth, dstheight, Bitmap.Config.ARGB_8888);
            drawBackground(bitmap);
            mCurrentPage.render(bitmap, null, null, PdfRenderer.Page.RENDER_MODE_FOR_DISPLAY);
            mCurrentPage.close();
            mCurrentPage = null;
            if (mPdfModel.mCallback != null) {
                mPdfModel.mCallback.loadData(null);
            }
            callback.onDataReady(bitmap);
        }
    }

    private void drawBackground(Bitmap bitmap) {
        Canvas canvas = new Canvas(bitmap);
        canvas.drawColor(Color.WHITE);
    }

    private Point getDispaly(Context context) {
        Point point = new Point();
        WindowManager windowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
        windowManager.getDefaultDisplay().getRealSize(point);
        return point;
    }


    @Override
    public void cleanup() {
        UpimeLog.i("PDF加载", " cleanup");
    }

    @Override
    public void cancel() {
        UpimeLog.i("PDF加载", " cancel");
    }

    @NonNull
    @Override
    public Class<Bitmap> getDataClass() {
        return Bitmap.class;
    }

    @NonNull
    @Override
    public DataSource getDataSource() {
        return DataSource.LOCAL;
    }
}

```

定义好ModelLoader之后就可以调用Glide加载了

```java
Glide.with(mContext)
     .load(new PdfModel(mContext, mRenderer, String.format(PDF_FORMAT_URI, position), pdfPath, mPdfCallback))
     .into(view);
```



