---
title: apk加固原理
date: 2018-02-27 10:04:48
tags: Android
subtitle: Android
categories: Android
---

本文主要通过学习姜维大神的[Android中的Apk的加固(加壳)原理解析和实现](http://www.wjdiankong.cn/android%e4%b8%ad%e7%9a%84apk%e7%9a%84%e5%8a%a0%e5%9b%ba%e5%8a%a0%e5%a3%b3%e5%8e%9f%e7%90%86%e8%a7%a3%e6%9e%90%e5%92%8c%e5%ae%9e%e7%8e%b0/)和雪一梦大神的[根据”so劫持”过360加固详细分析](https://bbs.pediy.com/thread-223796.htm) 来记录自己的学习过程。

> ### 为什么要对APK加固

目前的软件发展速度非常快，尤其是移动端软件越来越深入人们的生活之中，我们时时刻刻都需要这些软件来进行支付，查询以及浏览信息。我们的隐私信息以及经济安全可谓是都在app之中。所以这也加快了Apk加固技术的发展，加固后的apk虽然增大了不法分子的破解难度，但是也明显降低了app的运行效率，所以有一些加固是针对于apk的主要逻辑代码进行的。加固也分为dex加固和so加固，但是呢，dex加固的重要性应该是更加高一点的，因为反编译后的java代码的可读性更高。

> ### 加固原理

图片盗用姜维大神

![](http://ousaim1qx.bkt.clouddn.com/download.png)

从上图我们可以明白，apk加固主要步骤为：

将一个需要加固的apk文件，一个负责解密的壳程序，先将apk文件进行加密，然后将解密的壳程序与之合并，得到新的dex文件，再将壳的dex文件替换，并得到一个全新的apk文件。将某个被加固的apk解压，如下图：

![assets文件夹](http://ousaim1qx.bkt.clouddn.com/jiagua.png)

经过加密后的文件的assets文件夹中多出了两个libjiagu.so和libjiagu_x86.so文件，这就是apk被加固的标志。

![dex文件](http://ousaim1qx.bkt.clouddn.com/jadx.png)

我们用jadx打开加压后的dex文件，看到和加固的原理一样，dex文件被替换。以上就是加固的原理。

> ### Dex文件解析

这里我需要盗用一张非虫大神的图：

![Android dex文件格式结构图](http://ousaim1qx.bkt.clouddn.com/Android%20dex%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E7%BB%93%E6%9E%84%E5%9B%BE.png)

Dex是Android平台上(Dalvik虚拟机)的可执行文件。其实所有的加固，加密措施都是针对DexHeader中的checksum，signature和filesize来进行的，因为加固需要对dex文件进行改动，所以dex文件校验的数据也必须进行改动，知道了这个点之后，也就明白了，我们只要掌握dexheader其他的也就万变不离其宗了。

首先根据图中所标注的，前8个字节是整个dex文件的magic（魔法数），其中低地址的4个字节的数据是dex文件

的标识符，所有的dex文件的标志符都是一样的。高地址的4个字节数据是dex文件的版本，图中的dex文件版本是035。`0x08h-0x0Bh`这段地址上存放的是checksum，使用adler32加密算法，用来检验dex文件除magic和checksum以外的所有文件区域，检查错误。`0x0Ch-0x1Fh`这段地址上存放的是signature，使用SHA-1 hash算法，识别dex文件的唯一性。使用双重校验保证文件的安全性以及效率。`0x20h-0x23h`这段地址上存放的是这个dex文件的大小，`0x24h-0x27h`这段地址存放的是dexheader的大小，我发现好像所有的dexheader的大小都一样都是0x70，`0x28h-0x2Bh`这段地址上标志的是dex文件的字节序，像图中的0x78563412就是默认为小尾模式，跟C/C++中的小端模式一样**即高地址存放低字节，低地址存放高字节**，`0x2ch-0x2fh`中说明的是连接段的大小，默认为0表示静态连接。

> ### Apk加固具体步骤

首先我们需要一个需要加密的APK程序，在此我们就简单的写一个：

主Application：

```java
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i("mmm","被加密APK载入");
    }
}
```

这个Application很简单就是打印一下Log。

接下来是主Activity：

```java
public class MainActivity extends AppCompatActivity {
    private TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView=findViewById(R.id.text);
        textView.setText("被加密的App主页面");
    }
}
```

也是非常的简单，就是一个textview的展示。

按照上面的流程我们需要对源APK进行加密，那么我们就需要一个加密的程序：

```java
//加密的程序
public class MyMain {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		try {
            //读取源Apk
			File payloadSrcFile=new File("force/ForceApkObj.apk");
			//读取脱壳程序的dex
			File unShellDexFile=new File("force/ForceApk.dex");
            //将这个文件先转换为Byte，然后进行加密
			byte[] payloadArray=encrpt(readFileBytes(payloadSrcFile));
            //脱壳程序的dex也是一样
			byte[] unShellDexArray=readFileBytes(unShellDexFile);
            //分别得到这两个文件的大小，并计算出他们的总大小，最后加上的那4个字节存放的是目标APK的大小
			int payloadLen=payloadArray.length;
			int unShellDexLen=unShellDexArray.length;
			int totalLen=payloadLen+unShellDexLen+4;
			//申请我们所需要的byte数组，大小就是上面的总量
			byte[] newdex=new byte[totalLen];
			//先将脱壳的dex写进我们所申请的数组当中，其次是被加密的APK，最后是目标Apk的大小
			System.arraycopy(unShellDexArray, 0, newdex, 0, unShellDexLen);
			
			System.arraycopy(payloadArray, 0, newdex, unShellDexLen, payloadLen);
			
			System.arraycopy(intToByte(payloadLen),0, newdex, totalLen-4, 4);
			//分别修改dex文件大小，checksum和签名
			fixFileSizeHeader(newdex);
			
			fixSHA1Header(newdex);
			
			fixCheckSumHeader(newdex);
			
			//新创建一个dex文件，将我们完成的byte数组写入进去，这个dex也就是一个全新的dex
			String str="force/classes.dex";
			File file =new File(str);
			if (!file.exists()) {
				file.createNewFile();
			}
			FileOutputStream localFileOutputStream=new FileOutputStream(str);
			localFileOutputStream.write(newdex);
			localFileOutputStream.flush();
			localFileOutputStream.close();
			
		} catch (Exception e) {
			// TODO: handle exception
		}

	}
	
    //这是apk的加密算法，使用每个byte异或一下
	private static byte[] encrpt(byte[] srcdata) {
		for(int i=0;i<srcdata.length;i++) {
			srcdata[i]=(byte)(0xFF^srcdata[i]);
		}
		return srcdata;
	}
	
	//checksum
	private static void fixCheckSumHeader(byte[] dexBytes) {
		Adler32 adler32=new Adler32();
        //一开始我没有搞的很清楚，通过重看checksum明白了，checksum使用的是Adler32算法，他是对整个dex文
        //件除了前8个字节的魔法数和4个字节自己以外进行文件的错误检查的，在这里我们使用Adler32对象update
        //方法重新生成checksum。
		adler32.update(dexBytes, 12, dexBytes.length-12);
		long value=adler32.getValue();
		//拿到的checksum
		int va=(int)value;
		byte[] newcs=intToByte(va);
		
		byte[] recs=new byte[4];
        //在之前已经知道了，dex文件是采用小端模式，即高字节放低地址，低字节放字高地址上
		for(int i=0;i<4;i++) {
			recs[i]=newcs[newcs.length-1-i];
			System.out.println(Integer.toHexString(newcs[i]));
		}
        //对dex的checksum进行修改
		System.arraycopy(recs, 0, dexBytes, 8, 4);
	}
	
    //int转换成Byte
	public static byte[] intToByte(int number) {
		byte[] b=new byte[4];
		for(int i=3;i>=0;i--) {
			b[i]=(byte)(number%256);
			number>>=8;
		}
		return b;
	}
	//修改dex文件的签名
	private static void fixSHA1Header(byte[] dexBytes) throws 
	NoSuchAlgorithmException{
        //dex文件的签名方式使用SHA-1加密
		MessageDigest mDigest=MessageDigest.getInstance("SHA-1");
		mDigest.update(dexBytes, 32, dexBytes.length-32);
		byte[] newdt=mDigest.digest();
		System.arraycopy(newdt, 0, dexBytes, 12, 20);
	}
	
    //修改dex头文件的中的file_size
	private static void fixFileSizeHeader(byte[] dexBytes) {
		byte[] newfs=intToByte(dexBytes.length);
		byte[] refs=new byte[4];
		for(int i=0;i<4;i++) {
			refs[i]=newfs[newfs.length-1-i];
		}
		System.arraycopy(refs, 0, dexBytes, 32, 4);
	}
	
    //将源APk转换为Byte数组
	private static byte[] readFileBytes(File file) throws IOException{
		byte[] arrayOfByte=new byte[1024];
		ByteArrayOutputStream localByteArrayOutputStream =new ByteArrayOutputStream();
		FileInputStream fis=new FileInputStream(file);
		while(true) {
			int i=fis.read(arrayOfByte);
			if (i!=-1) {
				localByteArrayOutputStream.write(arrayOfByte,0,i);
			}else {
				return localByteArrayOutputStream.toByteArray();
			}
		}
	}
	

}
```

现在想一想，现在合并之后的Apk是如何找到源Apk进行加载的呢。因为类的加载都是通过classloader来进行加载的，那是不是可以说，我们只需要更改脱壳程序中的classloader就可以让他来加载源Apk程序了，那么这里又产生了一个问题，那就是他只会加载源程序，但是脱壳程的逻辑并不会执行，那这样就会导致源程序无法被解密，这样运行也是失败的，所以我们需要使这个加载源程序的dexclassloader以原来的classloader为父节点，那么这样脱壳程序也能顺利的执行并对源程序解密。

脱壳程序（这里我直接拿四哥的源码，在他基础上我继续分析一下）：

```java
/**这个是脱壳程序的application，主要做了两个重要的部分，1首先对源加密的apk进行解密。2使用动态加载来加载
*源APK的application让他执行自己的生命周期，因为attachBaseContext()他要比oncreate()执行靠前，所以源apk
*的解密工作就需要在这个方法中做。首先从Apk中拿到合并后的dex文件，然后根据尾部的大小，将源APK取出来，这时*候，apk依然是byte数组，然后我们对他进行解密操作，另外自定义dexclassloader，来加载源APK。在脱壳程序中的
*oncreate方法中主要是通过反射将当前的Application替换为源程序的Application。其实在ActivityThread中有一*个内部类，AppBindData,这个类里有启动的app的所有详细信息，我们只需要将这个类里的属性值通过反射进行置换就*可以了。
*/
public class ProxyApplication extends Application{  
    private static final String appkey = "APPLICATION_CLASS_NAME";  
    private String apkFileName;  
    private String odexPath;  
    private String libPath;  
  
    //这是context 赋值  
    @Override  
    protected void attachBaseContext(Context base) {  
        super.attachBaseContext(base);  
        try {  
            //创建两个文件夹payload_odex，payload_lib 私有的，可写的文件目录  
            File odex = this.getDir("payload_odex", MODE_PRIVATE);  
            File libs = this.getDir("payload_lib", MODE_PRIVATE);  
            odexPath = odex.getAbsolutePath();  
            libPath = libs.getAbsolutePath();  
            apkFileName = odex.getAbsolutePath() + "/payload.apk";  
            File dexFile = new File(apkFileName);  
            Log.i("demo", "apk size:"+dexFile.length());  
            if (!dexFile.exists())  
            {  
                dexFile.createNewFile();  //在payload_odex文件夹内，创建payload.apk  
                // 读取程序classes.dex文件  
                byte[] dexdata = this.readDexFileFromApk();  
                  
                // 分离出解壳后的apk文件已用于动态加载  
                this.splitPayLoadFromDex(dexdata);  
            }  
            // 配置动态加载环境  
            Object currentActivityThread = RefInvoke.invokeStaticMethod(  
                    "android.app.ActivityThread", "currentActivityThread",  
                    new Class[] {}, new Object[] {});//获取主线程对象 http://blog.csdn.net/myarrow/article/details/14223493  
            String packageName = this.getPackageName();//当前apk的包名  
            //下面两句不是太理解  
            ArrayMap mPackages = (ArrayMap) RefInvoke.getFieldOjbect(  
                    "android.app.ActivityThread", currentActivityThread,  
                    "mPackages");  
            WeakReference wr = (WeakReference) mPackages.get(packageName);  
            //创建被加壳apk的DexClassLoader对象  加载apk内的类和本地代码（c/c++代码）  
            DexClassLoader dLoader = new DexClassLoader(apkFileName, odexPath,  
                    libPath, (ClassLoader) RefInvoke.getFieldOjbect(  
                            "android.app.LoadedApk", wr.get(), "mClassLoader"));  
            //base.getClassLoader(); 是不是就等同于 (ClassLoader) RefInvoke.getFieldOjbect()? 有空验证下//?  
            //把当前进程的DexClassLoader 设置成了被加壳apk的DexClassLoader  ----有点c++中进程环境的意思~~  
            RefInvoke.setFieldOjbect("android.app.LoadedApk", "mClassLoader",  
                    wr.get(), dLoader);  
              
            try{  
                Object actObj = dLoader.loadClass("com.example.forceapkobj.MainActivity");  
                Log.i("demo", "actObj:"+actObj);  
            }catch(Exception e){  
                Log.i("demo", "activity:"+Log.getStackTraceString(e));  
            }  
              
  
        } catch (Exception e) {  
            Log.i("demo", "error:"+Log.getStackTraceString(e));  
            e.printStackTrace();  
        }  
    }  
  
    @Override  
    public void onCreate() {  
        {  
            //loadResources(apkFileName);  
              
            Log.i("demo", "onCreate");  
            // 如果源应用配置有Appliction对象，则替换为源应用Applicaiton，以便不影响源程序逻辑。  
            String appClassName = null;  
            try {  
                ApplicationInfo ai = this.getPackageManager()  
                        .getApplicationInfo(this.getPackageName(),  
                                PackageManager.GET_META_DATA);  
                Bundle bundle = ai.metaData;  
                if (bundle != null && bundle.containsKey("APPLICATION_CLASS_NAME")) {  
                    appClassName = bundle.getString("APPLICATION_CLASS_NAME");//className 是配置在xml文件中的。  
                } else {  
                    Log.i("demo", "have no application class name");  
                    return;  
                }  
            } catch (NameNotFoundException e) {  
                Log.i("demo", "error:"+Log.getStackTraceString(e));  
                e.printStackTrace();  
            }  
            //有值的话调用该Applicaiton  
            Object currentActivityThread = RefInvoke.invokeStaticMethod(  
                    "android.app.ActivityThread", "currentActivityThread",  
                    new Class[] {}, new Object[] {});  
            Object mBoundApplication = RefInvoke.getFieldOjbect(  
                    "android.app.ActivityThread", currentActivityThread,  
                    "mBoundApplication");  
            Object loadedApkInfo = RefInvoke.getFieldOjbect(  
                    "android.app.ActivityThread$AppBindData",  
                    mBoundApplication, "info");  
            //把当前进程的mApplication 设置成了null  
            RefInvoke.setFieldOjbect("android.app.LoadedApk", "mApplication",  
                    loadedApkInfo, null);  
            Object oldApplication = RefInvoke.getFieldOjbect(  
                    "android.app.ActivityThread", currentActivityThread,  
                    "mInitialApplication");  
            //http://www.codeceo.com/article/android-context.html  
            ArrayList<Application> mAllApplications = (ArrayList<Application>) RefInvoke  
                    .getFieldOjbect("android.app.ActivityThread",  
                            currentActivityThread, "mAllApplications");  
            mAllApplications.remove(oldApplication);//删除oldApplication  
              
            ApplicationInfo appinfo_In_LoadedApk = (ApplicationInfo) RefInvoke  
                    .getFieldOjbect("android.app.LoadedApk", loadedApkInfo,  
                            "mApplicationInfo");  
            ApplicationInfo appinfo_In_AppBindData = (ApplicationInfo) RefInvoke  
                    .getFieldOjbect("android.app.ActivityThread$AppBindData",  
                            mBoundApplication, "appInfo");  
            appinfo_In_LoadedApk.className = appClassName;  
            appinfo_In_AppBindData.className = appClassName;  
            Application app = (Application) RefInvoke.invokeMethod(  
                    "android.app.LoadedApk", "makeApplication", loadedApkInfo,  
                    new Class[] { Boolean.class, Instrumentation.class },  
                    new Object[] { false, null });//执行 makeApplication（false,null）  
            RefInvoke.setFieldOjbect("android.app.ActivityThread",  
                    "mInitialApplication", currentActivityThread, app);  
  
  
           
            //这里我还有点搞不明白。我猜应该是，将原来的provider替换为源apk的，这样才能保证程序的正常
            ArrayMap mProviderMap = (ArrayMap) RefInvoke.getFieldOjbect(  
                    "android.app.ActivityThread", currentActivityThread,  
                    "mProviderMap");  
            Iterator it = mProviderMap.values().iterator();  
            while (it.hasNext()) {  
                Object providerClientRecord = it.next();  
                Object localProvider = RefInvoke.getFieldOjbect(  
                        "android.app.ActivityThread$ProviderClientRecord",  
                        providerClientRecord, "mLocalProvider");  
                RefInvoke.setFieldOjbect("android.content.ContentProvider",  
                        "mContext", localProvider, app);  
            }  

              
            app.onCreate();  
        }  
    }  
  
    /** 
     * 释放被加壳的apk文件，so文件 
     * @param data 
     * @throws IOException 
     */  
    private void splitPayLoadFromDex(byte[] apkdata) throws IOException {  
        int ablen = apkdata.length;  
        //取被加壳apk的长度   这里的长度取值，对应加壳时长度的赋值都可以做些简化  
        byte[] dexlen = new byte[4];  
        System.arraycopy(apkdata, ablen - 4, dexlen, 0, 4);  
        ByteArrayInputStream bais = new ByteArrayInputStream(dexlen);  
        DataInputStream in = new DataInputStream(bais);  
        int readInt = in.readInt();  
        System.out.println(Integer.toHexString(readInt));  
        byte[] newdex = new byte[readInt];  
        //把被加壳apk内容拷贝到newdex中  
        System.arraycopy(apkdata, ablen - 4 - readInt, newdex, 0, readInt);  
        //这里应该加上对于apk的解密操作，若加壳是加密处理的话  
        //?  
          
        //对源程序Apk进行解密  
        newdex = decrypt(newdex);  
          
        //写入apk文件     
        File file = new File(apkFileName);  
        try {  
            FileOutputStream localFileOutputStream = new FileOutputStream(file);  
            localFileOutputStream.write(newdex);  
            localFileOutputStream.close();  
        } catch (IOException localIOException) {  
            throw new RuntimeException(localIOException);  
        }  
          
        //分析被加壳的apk文件  
        ZipInputStream localZipInputStream = new ZipInputStream(  
                new BufferedInputStream(new FileInputStream(file)));  
        while (true) {  
            ZipEntry localZipEntry = localZipInputStream.getNextEntry();//不了解这个是否也遍历子目录，看样子应该是遍历的  
            if (localZipEntry == null) {  
                localZipInputStream.close();  
                break;  
            }  
            //取出被加壳apk用到的so文件，放到 libPath中（data/data/包名/payload_lib)  
            String name = localZipEntry.getName();  
            if (name.startsWith("lib/") && name.endsWith(".so")) {  
                File storeFile = new File(libPath + "/"  
                        + name.substring(name.lastIndexOf('/')));  
                storeFile.createNewFile();  
                FileOutputStream fos = new FileOutputStream(storeFile);  
                byte[] arrayOfByte = new byte[1024];  
                while (true) {  
                    int i = localZipInputStream.read(arrayOfByte);  
                    if (i == -1)  
                        break;  
                    fos.write(arrayOfByte, 0, i);  
                }  
                fos.flush();  
                fos.close();  
            }  
            localZipInputStream.closeEntry();  
        }  
        localZipInputStream.close();  
  
  
    }  
  
    /** 
     * 从apk包里面获取dex文件内容（byte） 
     * @return 
     * @throws IOException 
     */  
    private byte[] readDexFileFromApk() throws IOException {  
        ByteArrayOutputStream dexByteArrayOutputStream = new ByteArrayOutputStream();  
        ZipInputStream localZipInputStream = new ZipInputStream(  
                new BufferedInputStream(new FileInputStream(  
                        this.getApplicationInfo().sourceDir)));  
        while (true) {  
            ZipEntry localZipEntry = localZipInputStream.getNextEntry();  
            if (localZipEntry == null) {  
                localZipInputStream.close();  
                break;  
            }  
            if (localZipEntry.getName().equals("classes.dex")) {  
                byte[] arrayOfByte = new byte[1024];  
                while (true) {  
                    int i = localZipInputStream.read(arrayOfByte);  
                    if (i == -1)  
                        break;  
                    dexByteArrayOutputStream.write(arrayOfByte, 0, i);  
                }  
            }  
            localZipInputStream.closeEntry();  
        }  
        localZipInputStream.close();  
        return dexByteArrayOutputStream.toByteArray();  
    }  
  
  
    // //直接返回数据，读者可以添加自己解密方法  
    private byte[] decrypt(byte[] srcdata) {  
        for(int i=0;i<srcdata.length;i++){  
            srcdata[i] = (byte)(0xFF ^ srcdata[i]);  
        }  
        return srcdata;  
    }  
      
      
    //以下是加载资源  
    protected AssetManager mAssetManager;//资源管理器    
    protected Resources mResources;//资源    
    protected Theme mTheme;//主题    
      
    protected void loadResources(String dexPath) {    
        try {    
            AssetManager assetManager = AssetManager.class.newInstance();    
            Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);    
            addAssetPath.invoke(assetManager, dexPath);    
            mAssetManager = assetManager;    
        } catch (Exception e) {    
            Log.i("inject", "loadResource error:"+Log.getStackTraceString(e));  
            e.printStackTrace();    
        }    
        Resources superRes = super.getResources();    
        superRes.getDisplayMetrics();    
        superRes.getConfiguration();    
        mResources = new Resources(mAssetManager, superRes.getDisplayMetrics(),superRes.getConfiguration());    
        mTheme = mResources.newTheme();    
        mTheme.setTo(super.getTheme());  
    }    
      
    @Override    
    public AssetManager getAssets() {    
        return mAssetManager == null ? super.getAssets() : mAssetManager;    
    }    
      
    @Override    
    public Resources getResources() {    
        return mResources == null ? super.getResources() : mResources;    
    }    
      
    @Override    
    public Theme getTheme() {    
        return mTheme == null ? super.getTheme() : mTheme;    
    }   
      
} 
```

最后，我们就把脱壳程序中的dex文件替换为合并之后的dex文件，再重新签个名。但是我走到这一步，总是出现安装失败的问题，我猜应该是重新签名索引发的问题，我将继续实验。