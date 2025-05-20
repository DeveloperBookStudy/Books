# Hilt

안드로이드에서 Hilt는 의존성 주입(Dependency Injection, DI)을 쉽게 할 수 있도록 도와주는 Jetpack 라이브러리입니다. 내부적으로는 Google의 Dagger를 기반으로 동작하지만, Hilt는 Dagger보다 더 간편한 설정과 사용법을 제공합니다.

 - @HiltAndroidApp: 애플리케이션 클래스에 붙여서 Hilt를 초기화
 - @AndroidEntryPoint: 의존성을 주입받을 수 있는 안드로이드 컴포넌트에 붙임 (Activity, Fragment 등)
 - @Inject: 생성자나 필드에 의존성 주입
 - @Module, @InstallIn: 의존성을 제공하는 방법 정의
 - @Provides, @Binds: 객체 제공 방법 정의

