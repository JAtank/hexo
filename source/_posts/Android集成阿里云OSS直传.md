---
title: Android集成阿里云OSS直传
date: 2017-10-30 21:16:19
tags:
---
首先想说下这个功能的背景，因为最近公司的项目维护升级，需要一个在webview调用照相机，直接拍照上传的功能。为了减少接口和业务服务器压力，所以选择将图片直传到阿里云上，所以自己开始摸索。

先说一下功能实现的步骤和遇到的几个问题：
1. 在webview中调用照相设备，获取图片。此步骤中的问题如下：
	（1）通过什么样的方法进行webview调用照相设备的操作。
	（2）在通过照相机照完照片后怎样获取照片。
	（3）照片该怎样压缩然后进行上传。
	（4）上传后怎样删除保存后的图片。
2. 通过阿里云进行图片上传。此步骤中的问题如下：
	（1）阿里云oss上传服务的集成。
	（2）阿里云有两种方式进行上传，该选择哪个？
	（3）webview（前端）该怎样和Android进行交互进行上传。

----------

既然存在问题，那我们就开始先解决问题，再进行实际操作：

*通过什么样的方法进行webview调用照相设备的操作*；*在通过照相机照完照片后怎样获取照片*。
这两个问题为什么放在一起，是因为他们的实现方式是连贯的，在某些地方需要相同的操作。这个两个问题最直观的解决方法，大家可能想到通过input标签进行调用，然后开启照相设备，然后进行图片选择。但是通过这个方法调用照相设备和获取照片是有限制的，就是在你的webview中的setWebChromeClient()方法中重写openFileChooser()方法，但是这里要注意，不同的android版本需要区分开。在我们的项目中需要的是在拍照后直接上传，不需要再进行选择，所以没有采用这种方式。我采取的方法是：通过js，调用android方法，直接打开照相设备（在android中打开照相设备比上述方法简单太多），然后通过指定具体的路径进行图片保存，并且在Android中直接获拿图片路径进行上传（这样获取图片路径的步骤就省了，因为图片路径已经指定）。
*照片该怎样压缩然后进行上传*。
这个其实就是图片压缩的问题，由于我们的图片对尺寸不做限制，所以我只进行了图片的质量压缩，方式很简单，将图片转换为bitmap格式，然后进行压缩，这应该是android最常用的压缩方式了。但这里我将图片保存转换时，将图片转换为了webp格式，为什么用这个格式是因为，webp图片的大小很小，占用内存很少，不管是在存的时候，还是在加载的时候都会提高效率，节省资源。这里可以给大家大致举例一下我的图片在转换压缩前后的对比：一张大概在1.4M的png图片，在压缩比为30%的时候，压缩后的图片大小在300k左右（大家没看错，实际的图片大小是比换算的更小），但在转化为webp后，只有60~70k，可以看到差距很明显。
*上传后怎样删除保存后的图片*。
为什么要删除，是为了防止设备常年使用，积累过多图片。删除是很简单的，上边已获得了图片路径，所以一般的文件删除即可。但是我想说的是，由于我们上传的图片并不是隐私或者重要的图片，所以我没有进行删除，而是在保存的时候进行了替换，这是一个投机取巧的方式，看大家具体的项目需求。
*阿里云oss上传服务的集成*。
这个问题我列出来是因为大家在看阿里云oss安卓直传的部分的时候,并没有写集成方式，所以还是需要按照正常的sdk集成，具体链接为：
[https://help.aliyun.com/document_detail/32042.htmlspm=5176.doc32043.6.696.vB2P2R](https://help.aliyun.com/document_detail/32042.html?spm=5176.doc32043.6.696.vB2P2R)
*阿里云有两种方式进行上传，该选择哪个*。
其实阿里云提供的上传方式大致可以分为两种，一种是直接携带固定的校验参数，一种是通过业务服务器获取动态凭证，为了安全性考虑，我采用的动态获取。
*webview（前端）该怎样和Android进行交互进行上传*。
答案很简单，调用通过js和安卓交互。具体的方式实现网上有很多，不在多说。我只说一下难点，我们的webview呈现的页面时通过vue.js这个前端框架编写的。在这里我们在上传开始的时候需要提示上传开始，上传完成后需要提示上传完成。这就要求我们要有事件监听。其实在安卓里我们有很多方式可以实现，比如：广播，EvenBus，接口设计模式……。但是这些不能用于前端中，我们在vue中，使用了$on,$off,$emit这几个指令来组合实现。


----------
具体的实现代码：
1. 集成oss：
   ![](https://i.imgur.com/R9lbwmF.png)
2. 然后根据文档进行方法封装，以下是我封装的方法:
<pre><code>
	import android.content.Context;
	import android.support.annotation.NonNull;
	import android.util.Log;
	import com.alibaba.sdk.android.oss.ClientConfiguration;
	import com.alibaba.sdk.android.oss.OSS;
	import com.alibaba.sdk.android.oss.OSSClient;
	import com.alibaba.sdk.android.oss.callback.OSSCompletedCallback;
	import com.alibaba.sdk.android.oss.common.auth.OSSCredentialProvider;
	import com.alibaba.sdk.android.oss.common.auth.OSSFederationCredentialProvider;
	import com.alibaba.sdk.android.oss.common.auth.OSSFederationToken;
	import com.alibaba.sdk.android.oss.internal.OSSAsyncTask;
	import com.alibaba.sdk.android.oss.model.PutObjectRequest;
	import com.alibaba.sdk.android.oss.model.PutObjectResult;
	import com.youhongedu.sx2.tea.v3.bean.UploadBean;
	import java.io.File;
	//阿里云上传工具类
	public class OssUploadUtil {
	    private static UploadBean mUploadBean;//升级javaBean
	    private static OssUploadUtil ossUploadUtil;
	    private static Context context;

	    private OssUploadUtil(Context context, UploadBean uploadBean){
	        this.context = context;
	        this.mUploadBean = uploadBean;
	    }
	    public static OssUploadUtil getOssUploadUtil(Context context,UploadBean uploadBean){
	        if(ossUploadUtil==null){
	            ossUploadUtil = new OssUploadUtil(context,uploadBean);
	        }
	        return ossUploadUtil;
	    }
	    // 初始化oss服务
	    public OSS initOSS() {
	        //使用自己的获取STSToken的类
	        OSSCredentialProvider credentialProvider = new STSGetter();
	        ClientConfiguration conf = new ClientConfiguration();
	        conf.setConnectionTimeout(15 * 1000); // 连接超时，默认15秒
	        conf.setSocketTimeout(15 * 1000); // socket超时，默认15秒
	        conf.setMaxConcurrentRequest(5); // 最大并发请求书，默认5个
	        conf.setMaxErrorRetry(2); // 失败后最大重试次数，默认2次
	        Log.e("endpoint",mUploadBean.getEndpoint());
	        OSS oss = new OSSClient(context, mUploadBean.getEndpoint(), credentialProvider, conf);
	        return oss;
	    }
	    //上传
	    public void asyncPutImage(String object, String localFile, OSS oss,
	                              @NonNull final OSSCompletedCallback<PutObjectRequest, PutObjectResult> userCallback) {
	        if (object.equals("")) {
	            Log.w("AsyncPutImage", "ObjectNull");
	            return;
	        }
	        File file = new File(localFile);
	        if (!file.exists()) {
	            Log.w("AsyncPutImage", "FileNotExist");
	            Log.w("LocalFile", localFile);
	            return;
	        }
	        Log.e("localFile",localFile+"-------------");
	        // 构造上传请求
	        Log.e("bucket",mUploadBean.getBucket());
	        PutObjectRequest put = new PutObjectRequest(mUploadBean.getBucket(), object, localFile);
	        OSSAsyncTask task = oss.asyncPutObject(put, userCallback);
	    }
	    //内部类：获取sts服务（同时携带临时token）
	    public class STSGetter extends OSSFederationCredentialProvider {
	        public STSGetter() {
	        }
	        public OSSFederationToken getFederationToken() {
	            Log.e("Accesskeyid",mUploadBean.getAccesskeyid()+"");
	            return new OSSFederationToken(mUploadBean.getAccesskeyid(), mUploadBean.getAccesskeysecret(),
	             mUploadBean.getSecuritytoken(), mUploadBean.getExpiration());
	        }
	    }
	}
3.安卓中调取照相机，在这里我们直接指定存储位置和名称
<pre><code>
    //拍照
      @JavascriptInterface
      public void take(){
          Log.e("take","点击成功");
          Intent cameraIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);//拍照
          Uri imageUri = Uri.fromFile(new File(Environment.getExternalStorageDirectory(),"/demo.jpg"));//直接指定存储位置和名称
          cameraIntent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
          startActivityForResult(cameraIntent,REQUEST_UPLOAD_FILE_CODE);
      }
4.获取bitmap图片，方便进行压缩
<pre><code>
   //获取bitmap格式
    public Bitmap getBitmap(String imgPath) {
       BitmapFactory.Options newOpts = new BitmapFactory.Options();
       newOpts.inJustDecodeBounds = false;
       newOpts.inPurgeable = true;
       newOpts.inInputShareable = true;
       newOpts.inSampleSize = 1;
       newOpts.inPreferredConfig = Bitmap.Config.RGB_565;
       return BitmapFactory.decodeFile(imgPath, newOpts);
    }
5.图片压缩处理
<pre><code>
   //  图片压缩
    private void compressPhoto(){
        Bitmap photo = getBitmap(Environment.getExternalStorageDirectory()+"/demo.jpg");
        FileOutputStream fos =null;
        try {
            fos = new FileOutputStream(Environment.getExternalStorageDirectory()+"/demo.webp");
            photo.compress(Bitmap.CompressFormat.WEBP,30,fos); //图片压缩
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }finally{
            try {
                fos.flush();
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
6.以上就是上传前的准备工作，由于此前封装好的工具类，在真正的上传过程中代码其实很简单
<pre><code>
 // 在OnActivityResult中处理拍照上传
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == REQUEST_UPLOAD_FILE_CODE && resultCode == RESULT_OK) {
            new Thread(new Runnable() {  //开启子线程防止照相后出现闪出桌面现象
                @Override
                public void run() {
                    compressPhoto();
                    OssUploadUtil ossUploadUtil = OssUploadUtil.getOssUploadUtil(WelcomeActivity.this,mUploadBean);
                    OSS oss = ossUploadUtil.initOSS();
                    ossUploadUtil.asyncPutImage(mUploadBean.getUpload_dir()+mUploadBean.getPhoto_name(),
                                                Environment.getExternalStorageDirectory()+"/demo.webp",oss,
                                                new OSSCompletedCallback<PutObjectRequest, PutObjectResult>() {
                        @Override
                        public void onSuccess(PutObjectRequest request, PutObjectResult result) {
							//这里是url的拼接（即图片在阿里云上的访问地址）
                            String url = "http://"+mUploadBean.getBucket()+"."+mUploadBean.getEndpoint()+
                            "/"+mUploadBean.getUpload_dir()+mUploadBean.getPhoto_name();
                            uploadUrl=url;
                            mUploadBean = null;
                     		//这里可进行上传后的操作
                            Log.e("success","成功+"+JSON.toJSONString(request));
                            Log.e("success","成功-"+result);
                            Log.e("url",url);
                        }
                        @Override
                        public void onFailure(PutObjectRequest request, ClientException clientException,
                                              ServiceException serviceException) {
                            mHandler.sendEmptyMessage(MSG_UNREGECT_ERROR);
                            mUploadBean = null;
                            Log.e("error","失败1:"+clientException);
                            Log.e("error","失败2:"+serviceException);
                            Log.e("error","失败0:"+JSON.toJSONString(request));
                        }
                    });
                }
            }).start();
        }
    }
这样整个上传工作都做完了，多说一句，在上传后返回的参数中并不包含图片在外网的访问地址，所以，需要手动拼接提交业务服务器。但是在网上我也看到有其他的获取url的方式，但我可能由于一些操作或者其他问题没有使用成功（并不代表别人的不能使用），所以大家可以自行研究。