# Retrofit

Retrofit은 Square에서 개발한 Android 및 Java용 타입 안전(type-safe) HTTP 클라이언트 라이브러리입니다.

 - Annotation 기반: 인터페이스에 메서드와 함께 @GET, @POST, @Body, @Query 등을 붙여 API 정의
 - 직렬화 지원: Gson, Moshi, Jackson 등을 이용해 JSON을 객체로 자동 변환
 - 비동기 / 동기 지원: Call.enqueue()로 비동기, Call.execute()로 동기 방식 호출 가능
 - OkHttp 기반: 내부적으로 OkHttp를 사용하므로, 강력한 네트워크 기능과 연동됨
 - 인터셉터 사용 가능: 헤더 추가, 로깅, 인증 등 중간 처리 가능
 - HTTP 통신 라이브러리 비교: https://oscarstory.tistory.com/72
    - ✅ Volley, Retrofit
        - 요청 시 : 자동으로 백그라운드 스레드에서 작업
        - 응답 후 : 자동으로 메인 스레드로 돌아가서 처리
    - ✅ OkHttp
        - 요청 시 : 자동으로 백그라운드 스레드에서 작업
        - 응답 후 : 자동으로 메인 스레드 전환이 안 되어서, 별도의 스레드 작업 필요
    - 메인 스레드 (UI 스레드): 화면(UI)을 그리는 스레드. 여기에 오래 걸리는 작업을 하면 앱이 "멈춘 것처럼" 보임 → ANR 에러 발생 가능.
    - 백그라운드 스레드: 오래 걸리는 작업(예: 네트워크 요청)은 여기서 해야 앱이 멈추지 않음.
```java
// Retrofit 사용 시
apiService.getMovies().enqueue(object : Callback<List<Movie>> {
    override fun onResponse(call: Call<List<Movie>>, response: Response<List<Movie>>) {
        // 이미 메인 스레드니까 UI 바로 수정 가능
        textView.text = "성공!"
    }

    override fun onFailure(call: Call<List<Movie>>, t: Throwable) {
        textView.text = "실패!"
    }
})

// OkHttp 사용 시
val request = Request.Builder().url("https://example.com").build()
client.newCall(request).enqueue(object : Callback {
    override fun onResponse(call: Call, response: Response) {
        // ⚠️ 여기선 백그라운드 스레드임
        // 메인(UI) 스레드에서 UI 변경하려면 다음처럼 해야 함
        runOnUiThread {
            textView.text = "성공!"
        }
    }

    override fun onFailure(call: Call, e: IOException) {
        runOnUiThread {
            textView.text = "실패!"
        }
    }
})
```

## 1. Retrofit 사용법

 - `build.gradle`
```groovy
dependencies {
    // Retrofit
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    
    // Gson 컨버터 (JSON 파싱용)
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    
    // OkHttp 로깅 인터셉터 (디버깅용, 선택사항)
    implementation 'com.squareup.okhttp3:logging-interceptor:4.9.1'
}
```

 - `AndroidManifest.xml`
```xml
<manifest ...>
    <uses-permission android:name="android.permission.INTERNET" />
    ...
</manifest>
```

 - `API 인터페이스 생성`
    - @Path: URL PathVariable 설정
    - @Query: URL QueryParam 설정
    - @Body: 요청 본문 설정
    - @FormUrlEncoded: 폼 데이터 전송시 @Field와 함께 사용
    - @Multipart: 파일 업로드 등 멀티파트 요청시 @Part와 함께 사용
    - @Headers: HTTP 헤더 설정
```kotlin
interface MovieApiService {

    @GET("movies")
    fun getMovies(): Call<List<Movie>>

    @GET("movies/{id}")
    fun getMovie(@Path("id") movieId: Int): Call<Movie>

    @POST("movies")
    fun createMovie(@Body movie: Movie): Call<Movie>

    @PUT("movies/{id}")
    fun updateMovie(@Path("id") movieId: Int, @Body movie: Movie): Call<Movie>

    @DELETE("movies/{id}")
    fun deleteMovie(@Path("id") movieId: Int): Call<Void>
}
```

 - `Retrofit 인스턴스 생성 및 설정`
```kotlin
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create()) // 응답 컨버터
    .build()

// 인터페이스 구현체 생성
val apiService = retrofit.create(MovieApiService::class.java)

// 비동기 API 호출
apiService.getMovies().enqueue(object : Callback<List<Movie>> {
    override fun onResponse(call: Call<List<Movie>>, response: Response<List<Movie>>) {
        if (response.isSuccessful) {
            val movies = response.body()
            updateUI(movies)
        } else {
            handleErrorResponse(response.code())
        }
    }

    override fun onFailure(call: Call<List<Movie>>, t: Throwable) {
        handleNetworkError(t)
    }
})
```

 - `Retrofit 고급 설정`
```kotlin
// 로깅 인터셉터 생성
HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor();
loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);

// OkHttp 클라이언트 생성
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(loggingInterceptor)    // 로깅 추가
    .connectTimeout(30, TimeUnit.SECONDS)  // 연결 타임아웃
    .readTimeout(30, TimeUnit.SECONDS)     // 읽기 타임아웃
    .writeTimeout(30, TimeUnit.SECONDS)    // 쓰기 타임아웃
    .build();

// Retrofit 인스턴스에 OkHttp 클라이언트 설정
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(client)  // 커스텀 클라이언트 설정
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```

 - `싱글톤 패턴 활용하기`
```kotlin
public class RetrofitClient {
    private static Retrofit retrofit = null;
    
    public static Retrofit getClient(String baseUrl) {
        if (retrofit == null) {
            retrofit = new Retrofit.Builder()
                .baseUrl(baseUrl)
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        }
        return retrofit;
    }
}
```

 - `코루틴과 Retrofit`
```kotlin
// API 인터페이스 (코틀린)
interface MovieApiService {
    @GET("movies")
    suspend fun getMovies(): List<Movie>
}

// API 호출 (코틀린 + 코루틴)
lifecycleScope.launch {
    try {
        val movies = apiService.getMovies()
        updateUI(movies)
    } catch (e: HttpException) {
        handleErrorResponse(e.code())
    } catch (e: Throwable) {
        handleNetworkError(e)
    }
}
```

## 2. 에러 핸들링

 - `네트워크 에러 처리`
```java
@Override
public void onFailure(Call<MovieResponse> call, Throwable t) {
    // 구체적인 에러 타입 확인
    if (t instanceof UnknownHostException) {
        // 인터넷 연결 문제
        showError("인터넷 연결을 확인해주세요.");
    } else if (t instanceof SocketTimeoutException) {
        // 타임아웃
        showError("서버 응답이 너무 늦어요. 나중에 다시 시도해주세요.");
    } else if (t instanceof JsonSyntaxException) {
        // JSON 파싱 오류
        showError("데이터 형식에 문제가 있어요.");
    } else {
        // 기타 오류
        showError("오류가 발생했어요: " + t.getMessage());
    }
    
    // 로그 기록
    Log.e("API_ERROR", "API 호출 실패", t);
}
```

 - `HTTP 에러 처리`
```java
@Override
public void onResponse(Call<MovieResponse> call, Response<MovieResponse> response) {
    if (response.isSuccessful()) {
        // 성공 처리
        handleSuccessResponse(response.body());
    } else {
        // HTTP 에러 처리
        switch (response.code()) {
            case 400:
                showError("잘못된 요청이에요.");
                break;
            case 401:
                showError("인증에 실패했어요. 다시 로그인해주세요.");
                refreshToken(); // 토큰 갱신 시도
                break;
            case 403:
                showError("접근 권한이 없어요.");
                break;
            case 404:
                showError("요청한 정보를 찾을 수 없어요.");
                break;
            case 500:
            case 502:
            case 503:
            case 504:
                showError("서버에 문제가 있어요. 잠시 후 다시 시도해주세요.");
                break;
            default:
                showError("오류가 발생했어요: " + response.code());
                break;
        }
        
        // 오류 응답 본문 로깅
        try {
            if (response.errorBody() != null) {
                Log.e("API_ERROR", "Error body: " + response.errorBody().string());
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

 - `재시도 메커니즘`
```java
// OkHttp 인터셉터를 사용한 재시도 메커니즘
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new Interceptor() {
        @Override
        public okhttp3.Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            okhttp3.Response response = chain.proceed(request);
            
            int retryCount = 0;
            while (!response.isSuccessful() && retryCount < 3) {
                retryCount++;
                Log.d("RETRY", "Retry #" + retryCount);
                
                // 이전 응답 닫기
                response.close();
                
                // 잠시 대기 (지수 백오프)
                try {
                    Thread.sleep(1000 * retryCount);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                
                // 재시도
                response = chain.proceed(request);
            }
            
            return response;
        }
    })
    .build();
```

## 3. 사용 예시

 - `동적 URL 및 헤더 설정`
```java
public interface ApiService {
    @GET
    Call<ResponseData> getDynamicUrl(@Url String fullUrl);
    
    @GET("users/{user}/repos")
    Call<List<Repo>> listRepos(
        @Path("user") String user,
        @Query("sort") String sort,
        @Query("page") int page
    );
    
    @GET("protected-resource")
    Call<SecretData> getProtectedResource(
        @Header("Authorization") String authToken
    );
    
    // 여러 헤더를 한번에 설정
    @Headers({
        "Accept: application/vnd.github.v3.full+json",
        "User-Agent: My-App"
    })
    @GET("users/{username}")
    Call<User> getUser(@Path("username") String username);
    
    // 동적 헤더 맵 사용
    @GET("some-resource")
    Call<Data> getData(@HeaderMap Map<String, String> headers);
}
```

 - `폼 데이터 및 멀티파트 요청`
```java
// 폼 데이터 전송
@FormUrlEncoded
@POST("user/login")
Call<LoginResponse> login(
    @Field("email") String email,
    @Field("password") String password
);

// 멀티파트 요청 (파일 업로드)
@Multipart
@POST("user/profile")
Call<ProfileResponse> updateProfile(
    @Part("name") RequestBody name,
    @Part MultipartBody.Part photo
);

// 파일 업로드 구현 예시
File file = new File(imagePath);
RequestBody requestFile = RequestBody.create(MediaType.parse("image/*"), file);
MultipartBody.Part photoPart = MultipartBody.Part.createFormData("photo", file.getName(), requestFile);

RequestBody nameBody = RequestBody.create(MediaType.parse("text/plain"), userName);

apiService.updateProfile(nameBody, photoPart).enqueue(new Callback<ProfileResponse>() {
    // 콜백 구현...
});
```

 - `인증 인터셉터`
```java
// 모든 요청에 인증 토큰을 자동으로 추가하는 인터셉터
public class AuthInterceptor implements Interceptor {
    private SessionManager sessionManager;
    
    public AuthInterceptor(SessionManager sessionManager) {
        this.sessionManager = sessionManager;
    }
    
    @Override
    public okhttp3.Response intercept(Chain chain) throws IOException {
        Request originalRequest = chain.request();
        
        // 인증이 필요한 요청인지 확인 (예: 특정 URL 패턴)
        if (needsAuth(originalRequest)) {
            String token = sessionManager.getAuthToken();
            if (token == null) {
                // 토큰이 없으면 로그인 화면으로 리다이렉트 등의 처리
                throw new IOException("Authentication required");
            }
            
            // 토큰을 헤더에 추가한 새 요청 생성
            Request newRequest = originalRequest.newBuilder()
                .header("Authorization", "Bearer " + token)
                .build();
            
            return chain.proceed(newRequest);
        }
        
        return chain.proceed(originalRequest);
    }
    
    private boolean needsAuth(Request request) {
        // 인증이 필요한 URL 패턴 확인 로직
        String url = request.url().toString();
        return !url.contains("/login") && !url.contains("/register");
    }
}

// OkHttpClient에 인터셉터 추가
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new AuthInterceptor(sessionManager))
    .build();
```

 - `커스텀 타입 컨버터`
```java
// 예: Date 객체를 특정 형식으로 변환하는 컨버터
public class DateConverter implements Converter<Date, String> {
    private final SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'", Locale.US);
    
    @Override
    public String convert(Date date) throws IOException {
        return format.format(date);
    }
}

// 커스텀 컨버터 팩토리 구현
public class DateConverterFactory extends Converter.Factory {
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, 
                                                         Annotation[] annotations, 
                                                         Retrofit retrofit) {
        if (type == Date.class) {
            return new Converter<Date, RequestBody>() {
                @Override
                public RequestBody convert(Date date) throws IOException {
                    DateConverter dateConverter = new DateConverter();
                    String dateString = dateConverter.convert(date);
                    return RequestBody.create(MediaType.parse("text/plain"), dateString);
                }
            };
        }
        return null;
    }
}

// Retrofit에 커스텀 컨버터 추가
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(new DateConverterFactory())
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```

 - `요청 취소하기`
```java
// Call 객체 저장
private Call<MovieResponse> call;

// API 호출
private void loadMovies() {
    // 이전 요청이 있으면 취소
    if (call != null) {
        call.cancel();
    }
    
    // 새 요청 생성 및 실행
    call = apiService.getPopularMovies(API_KEY);
    call.enqueue(new Callback<MovieResponse>() {
        // 콜백 구현...
    });
}

// 액티비티 종료 시 요청 취소
@Override
protected void onDestroy() {
    super.onDestroy();
    if (call != null) {
        call.cancel();
    }
}
```
