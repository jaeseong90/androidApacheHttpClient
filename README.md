# 안드로이드 Android org.apache.http 지원중단에 따른 오류 해결

####구글에 따르면 

<hr/>

동작 변경사항: API 레벨 28+를 타겟팅하는 앱 


Apache HTTP 클라이언트 지원 중단
Android 6.0에서는 Apache HTTP 클라이언트 지원 기능이 삭제되었습니다. Android 9부터는 이 라이브러리가 bootclasspath에서 제거되고 기본적으로 앱에서 사용할 수 없습니다.

Android 9 이상을 타겟팅하는 앱이 Apache HTTP 클라이언트를 계속 사용하려면 다음을 AndroidManifest.xml에 추가해야 합니다.

```
<uses-library android:name="org.apache.http.legacy" android:required="false"/>
```
    
참고: 23 이하의 최소 SDK가 있는 앱에는 android:required="false" 속성이 필요합니다. API 레벨이 24보다 낮은 기기에서는 org.apache.http.legacy 라이브러리를 사용할 수 없기 때문입니다. (이러한 기기에는 Apache HTTP 클래스가 bootclasspath에서 제공됩니다.)
런타임 Apache 라이브러리를 사용하는 대신 앱에서 APK에 org.apache.http 라이브러리의 자체 버전을 번들링할 수 있습니다. 이 방법을 사용하려면 런타임에서 제공되는 클래스와의 클래스 호환성 문제를 피하기 위해 Jar Jar 등의 유틸리티로 라이브러리를 다시 패키징해야 합니다.

- https://developer.android.com/about/versions/pie/android-9.0-changes-28#apache-p

<hr/>

#### 기존에 안드로이드 프로젝트 문제 
- sdk버전을 올리면서 Deprecated 된 부분이 많음.

```
public HashMap<String,String> requestMultiparmHttpWithUrl(String _file, String _url, String _fileId){ 
		
		String callUrl = "uploadChatFile.do";
		if ( _url != null ) {
			callUrl = _url;
		}
		
		HashMap<String,String> result =  null;

        File file = new File( _file );

        //파일을 서버로 보내는 부분
        HttpClient client = null;
        if ( PCConstans.BASE_URL.startsWith("https") ) {
        	client = getHttpClient();
		} else {
			client = new DefaultHttpClient();
		}

        final HttpParams httpParameters = client.getParams();
        HttpConnectionParams.setConnectionTimeout(httpParameters, PCConstans.HTTP_CONNECT_TIMEOUT * 1000);

        HttpPost post = new HttpPost( PCConstans.BASE_URL + callUrl );

        FileBody bin  = new FileBody(file);
        
        MultipartEntityBuilder meb = MultipartEntityBuilder.create();
        meb.setCharset(Charset.forName("UTF-8"));
        meb.setMode(HttpMultipartMode.BROWSER_COMPATIBLE);
        meb.addPart("a", bin);
        meb.addTextBody("b", this.citizenInfo.getUserId());
        if ( _fileId != null && !"".equals(_fileId) ) {
        	meb.addTextBody("c", _fileId);
        }
        
        HttpEntity entity = meb.build();

        post.setEntity(entity);
		post.setHeader("Accept", "*/*");
		post.setHeader("Accpet-Language", "ko-KR");
		post.setHeader("User-Agent", "Mozilla/5.0 (Windows NT 6.3; WOW64; Trident/7.0; rv:11.0) like Gecko");

        try {            
            HttpResponse reponse = client.execute(post);
            result = writeStringFromStreem(reponse);
        } catch (Exception e){
            e.printStackTrace();
        } finally {
        }

        return result;
		
	}
  
  /* for ssl connection */
	private HttpClient getHttpClient() {

		try {

			KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
			trustStore.load(null, null);

			SSLSocketFactory sf = new SFSSLSocketFactory(trustStore);
			sf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);

			HttpParams params = new BasicHttpParams();
			HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
			HttpProtocolParams.setContentCharset(params, HTTP.UTF_8);

			SchemeRegistry registry = new SchemeRegistry();
			registry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
			registry.register(new Scheme("https", sf, 443));


			//ClientConnectionManager ccm = new ThreadSafeClientConnManager(registry);
			ClientConnectionManager ccm = new ThreadSafeClientConnManager(params, registry);

			return new DefaultHttpClient(ccm, params);

		} catch (Exception e) {
			return new DefaultHttpClient();
		}
	}
  
  private  HashMap<String,String> writeStringFromStreem(HttpResponse _reponse) {
		
		String strXml          = "";
        int httpResultCode     = HTTPRESULT_ERROR;
        
		InputStream             is = null;
        ByteArrayOutputStream baos = null;
        HashMap<String,String> result =  new HashMap<String,String>();
        
        
        try {
        	is = _reponse.getEntity().getContent();

            baos = new ByteArrayOutputStream();
            byte[] byteBuffer = new byte[1024];
            byte[] byteData   = null;
            int nLength = 0;

            while((nLength = is.read(byteBuffer, 0, byteBuffer.length)) != -1) {
                baos.write(byteBuffer, 0, nLength);
            }
            byteData = baos.toByteArray();
            strXml = new String(byteData);
            
            httpResultCode     = HTTPRESULT_OK;
            
        } catch (ClientProtocolException e) {
            e.printStackTrace();
            httpResultCode     = HTTPRESULT_NOT_FOUND;
        } catch (SocketTimeoutException e) {
            e.printStackTrace();
            httpResultCode     = HTTPRESULT_NOT_FOUND;
        } catch (ConnectTimeoutException e) {
            e.printStackTrace();
            httpResultCode     = HTTPRESULT_NO_SERVERINFO;
        } catch (IOException e) {
            e.printStackTrace();
            httpResultCode     = HTTPRESULT_ERROR;
        } finally {
        	
        	if ( baos != null ) { try { baos.close(); } catch (Exception e) {} }
        	if ( is != null ) { try { is.close(); } catch (Exception e) {} }
        	
        	result.put("CODE", ""+httpResultCode);
            result.put("VALUE", strXml);
        }
        
        return result;
	}
  
```

### 해결방안
- 기존 소스코드가 동작될 수 있도록 부분부분 수정 

build.gradle
```
android {
  useLibrary 'org.apache.http.legacy'
}
```

AndroidManifest.xml
```

 <application
        android:requestLegacyExternalStorage="true" // 이건 또 레거시 외부 스토리지 접근 다른 문제지만 아무튼 추가해주자
        android:usesCleartextTraffic="true" // https 를 기본값으로 사용함에 따른 대응
        >

        <!--
            2022.07.04
            According to Google, with Android 6.0, they removed support for the Apache HTTP client.
            Beginning with Android 9, that library is removed from the bootclasspath and is not available to apps by default.
            So if you still want to make it available to your application Google has given the solution for that also.
            Use the below tag inside application of your AndroidManifest.xml file
        -->
        <uses-library android:name="org.apache.http.legacy" android:required="false" />
</application>
