
title: "Glance 위젯이 프로세스 재시작 후 무한 로딩에 빠지는 이유 — AOSP 오픈소스 기여기" excerpt: "android/platform-samples의 WeatherGlanceWidget 버그를 재현·분석하고, GlanceAppWidget의 데이터 로딩 책임을 provideGlance로 옮겨 해결한 과정" categories:
* Android tags:
* Kotlin
* Glance
* AppWidget
* OpenSource
* Android toc: true toc_sticky: true date: 2026-08-19

⠀
# 개요
구글 공식 샘플 저장소인 ~[android/platform-samples](https://github.com/android/platform-samples)~를 살펴보다가 열려 있는 이슈 하나를 발견했습니다.
[Bug]: WeatherGlanceWidget not reloading after relaunch (#402)
Glance 기반 위젯인 WeatherGlanceWidget이 특정 실행 설정에서 로딩 상태에 갇힌 채 정상적으로 데이터를 표시하지 못한다는 내용이었습니다.
TagFileManager를 개발하며 Glance나 AppWidget의 생명주기를 깊게 다뤄본 적은 없었지만, 재현 방법이 명확하고 실제 코드까지 확인할 수 있는 버그였기 때문에 직접 원인을 분석하고 기여해보기로 했습니다.
이 글에서는 이슈를 재현하고, 로그와 AndroidX Glance 소스 코드를 따라가며 원인을 특정한 뒤, 공식 문서를 근거로 수정 방향을 결정하고 검증하기까지의 과정을 정리합니다.

# 문제 발견
이슈에 첨부된 재현 방법은 다음과 같았습니다.
* Android Studio Run/Debug Configuration에서 **Always install with package manager** 설정
* Launch Options를 **Nothing**으로 설정
* 기존 위젯이 홈 화면에 있는 상태에서 앱을 반복 실행

⠀문제가 발생한 코드에서는 GlanceAppWidgetReceiver의 onEnabled()에서 날씨 데이터를 갱신하고 있었습니다.
### @RequiresApi(Build.VERSION_CODES.O)
### class WeatherGlanceWidgetReceiver : GlanceAppWidgetReceiver() {
###     override val glanceAppWidget = WeatherGlanceWidget()

###     override fun onEnabled(context: Context?) {
###         super.onEnabled(context)
###         CoroutineScope(Dispatchers.IO).launch {
###             WeatherRepo.updateWeatherInfo()
###         }
###     }

###     override fun onUpdate(
###         context: Context,
###         appWidgetManager: AppWidgetManager,
###         appWidgetIds: IntArray,
###     ) {
###         super.onUpdate(context, appWidgetManager, appWidgetIds)
###     }
### }
문제 상황에서는 위젯이 시스템 기본 로딩 레이아웃인 glance_default_loading_layout을 표시한 채 정상적인 콘텐츠로 전환되지 않았습니다.
리사이즈를 수행하면 provideGlance()가 다시 호출되면서 위젯이 갱신되는 현상도 확인할 수 있었습니다.

# 원인 분석
먼저 정상적으로 동작하는 경우와 문제가 발생하는 경우의 Logcat을 비교했습니다.
### 정상 케이스
일반적인 Run으로 위젯을 추가했을 때는 다음과 같은 순서로 로그가 발생했습니다.
### onEnabled
### WeatherRepo: updateWeatherInfo called
### onUpdate for 1 widgets
### provideGlance
### Rendering Loading
### WeatherRepo: updateWeatherInfo - setting state to Available
### Rendering Available
### onEnabled()에서 updateWeatherInfo()가 호출되고, WeatherRepo의 상태가 Available로 변경된 뒤 위젯이 정상적으로 데이터를 표시합니다.
### 비정상 케이스
반면 문제를 재현했을 때는 다음과 같았습니다.
### onUpdate for 1 widgets
### provideGlance
### Rendering Loading
### Rendering Loading
가장 중요한 차이는 WeatherRepo: updateWeatherInfo called **로그가 존재하지 않았다는 것**입니다.
즉 provideGlance()나 onUpdate() 자체가 호출되지 않은 것이 아니라, **날씨 데이터를 갱신하는 로직이 호출되지 않고 있었습니다.**

# onEnabled()를 따라가 보자
그렇다면 왜 onEnabled()가 호출되지 않았을까요?
Glance의 Receiver는 결국 Android의 AppWidgetProvider lifecycle을 따릅니다.
공식 문서에서도 onEnabled()를 다음과 같이 설명합니다.
Triggered when the first instance of your widget is successfully created.
즉 onEnabled()는 **해당 Provider의 첫 번째 위젯 인스턴스가 생성될 때 호출되는 lifecycle callback**입니다.
따라서 이미 홈 화면에 위젯이 존재하는 상태에서 프로세스가 다시 시작된다고 해서 onEnabled()가 다시 호출되는 것은 아닙니다.
이번 문제를 단순화하면 다음과 같습니다.
### 첫 번째 위젯 추가
###     ↓
### onEnabled()
###     ↓
### WeatherRepo.updateWeatherInfo()
###     ↓
### WeatherInfo = Available
여기까지는 정상입니다.
하지만 이후 프로세스가 종료되고 다시 시작되면:
### 홈 화면의 위젯은 그대로 존재
###     ↓
### 앱 프로세스 재시작
###     ↓
### WeatherRepo의 인메모리 상태 초기화
###     ↓
### WeatherInfo = Loading
###     ↓
### onEnabled() 호출 ❌
###     ↓
### onUpdate()
###     ↓
### provideGlance()
###     ↓
### WeatherInfo = Loading
###     ↓
### 데이터 갱신 트리거 없음
이 구조에서는 WeatherRepo의 상태를 다시 Available로 만들어 줄 코드가 실행되지 않습니다.
특히 WeatherRepo의 상태는 MutableStateFlow를 기반으로 한 **인메모리 상태**이기 때문에 프로세스가 새로 시작되면 이전 상태를 그대로 유지하지 않습니다.
결국 이번 문제의 근본적인 원인은 다음 두 가지가 결합된 것이었습니다.
1. 데이터 상태는 프로세스 생명주기에 따라 초기화되는 인메모리 상태
2. 데이터를 초기화하는 updateWeatherInfo()는 첫 번째 위젯 생성 시에만 호출되는 onEnabled()에 의존

⠀"Always install with package manager"는 이 문제를 쉽게 재현할 수 있게 해주는 방법일 뿐, 근본적인 문제는 **위젯이 이미 존재하는 상태에서 프로세스가 다시 시작되었을 때 데이터 로딩을 보장하지 못하는 구조**였습니다.
실제 환경에서도 앱 업데이트, 기기 재부팅, 시스템에 의한 프로세스 종료 후 재시작 등 비슷한 상황이 발생할 수 있기 때문에 단순한 디버그 설정의 문제로만 볼 수 없었습니다.

# 시행착오 —onUpdate()에 추가하기
가장 먼저 생각한 해결책은 onUpdate()에서도 데이터를 갱신하는 것이었습니다.
### override fun onUpdate(
###     context: Context,
###     appWidgetManager: AppWidgetManager,
###     appWidgetIds: IntArray,
### ) {
###     super.onUpdate(context, appWidgetManager, appWidgetIds)

###     CoroutineScope(Dispatchers.IO).launch {
###         WeatherRepo.updateWeatherInfo()
###     }
### }
실제로 이 방법으로 문제는 해결됐습니다.
### onUpdate()가 호출될 때마다 날씨 데이터를 갱신하기 때문에 프로세스가 다시 시작되어 WeatherRepo의 상태가 초기화되더라도 다시 데이터를 가져올 수 있었습니다.
하지만 코드를 조금 더 살펴보니 의문이 생겼습니다.
### onUpdate()는 특정한 하나의 렌더링 작업을 의미하는 callback이 아닙니다.
위젯 업데이트 과정에서 다양한 이유로 호출될 수 있고, Receiver 수준에서 Provider의 위젯 업데이트 이벤트를 처리합니다.
따라서 단순히 onUpdate()에 데이터 로딩을 추가하는 것은 **현재 발생한 증상을 해결하기 위한 우회책**으로는 동작하지만, "위젯을 렌더링하는 데 필요한 데이터를 어디에서 준비해야 하는가?"라는 질문에는 명확한 답이 되지 않았습니다.
그래서 AndroidX Glance 내부 동작을 더 따라가 보기로 했습니다.

# Glance 내부 동작 추적
### GlanceAppWidgetReceiver.onUpdate()에서 시작해 실제 위젯 업데이트가 어떻게 진행되는지 확인했습니다.
### onUpdate()에서는 내부적으로 doUpdate()를 호출하고, 여기에서 각 위젯 ID에 대해 glanceAppWidget.update()가 실행됩니다.
### internal suspend fun CoroutineScope.doUpdate(
###     context: Context,
###     appWidgetIds: IntArray,
### ) {
###     updateManager(context)
###     appWidgetIds
###         .map { async { glanceAppWidget.update(context, it) } }
###         .awaitAll()
### }
그리고 update()에서는 해당 위젯의 AppWidgetSession을 가져오거나 생성합니다.
### internal suspend fun update(
###     context: Context,
###     appWidgetId: Int,
###     options: Bundle? = null,
### ) {
###     Tracing.beginGlanceAppWidgetUpdate()

###     val glanceId = AppWidgetId(appWidgetId)

###     getOrCreateAppWidgetSession(context, glanceId, options) { session, wasRunning ->
###         if (wasRunning) {
###             session.updateGlance()
###         }
###     }
### }
세션이 존재하지 않는 경우에는 createAppWidgetSession()을 통해 새로운 AppWidgetSession이 생성됩니다.
### protected open fun createAppWidgetSession(
###     context: Context,
###     id: AppWidgetId,
###     options: Bundle? = null,
### ): AppWidgetSession =
###     AppWidgetSession(this@GlanceAppWidget, id, options)
이 과정을 따라가면서 중요한 지점을 확인할 수 있었습니다.
### onUpdate()는 위젯 업데이트를 시작하는 Provider-level callback이고, 실제 Glance 콘텐츠를 제공하는 단계는 결국 GlanceAppWidget의 provideGlance()로 이어집니다.
그렇다면 **데이터를 로드해야 하는 위치 역시 이 지점이 더 적절하지 않을까?**라는 의문이 생겼습니다.

# 근본 해결 —provideGlance()에서 데이터 로딩
이때 Glance 공식 문서의 provideGlance() KDoc을 확인했습니다.
"This is a good place to load any data needed to render the Composable. Use provideContent to provide the Composable once the data is ready."
내용을 그대로 해석하면,
**Composable을 렌더링하는 데 필요한 데이터를** provideGlance()**에서 로드하고, 데이터가 준비된 후** provideContent()**를 호출하는 것을 권장한다**는 의미입니다.
현재 코드의 구조는 다음과 같았습니다.
### override suspend fun provideGlance(
###     context: Context,
###     id: GlanceId,
### ) {
###     provideContent {
###         Content()
###     }
### }
따라서 데이터 로딩을 provideGlance()의 데이터 준비 단계로 이동했습니다.
### @RequiresApi(Build.VERSION_CODES.O)
### override suspend fun provideGlance(
###     context: Context,
###     id: GlanceId,
### ) {
###     WeatherRepo.updateWeatherInfo()
###     provideContent {
###         Content()
###     }
### }
그리고 Receiver의 onEnabled()와 onUpdate()에서 별도로 실행하던 updateWeatherInfo() 호출은 제거했습니다.
이제 데이터 흐름은 다음과 같이 단순해집니다.
### 위젯 업데이트
###     ↓
### provideGlance()
###     ↓
### WeatherRepo.updateWeatherInfo()
###     ↓
### 데이터 준비
###     ↓
### provideContent()
###     ↓
### Content()
데이터를 준비하는 단계와 콘텐츠를 제공하는 단계가 명확하게 분리됩니다.

# 왜LaunchedEffect가 아니라 provideGlance()인가?
처음에는 다음과 같이 Content() 내부에서 데이터를 갱신하는 방법도 생각할 수 있습니다.
### LaunchedEffect(Unit) {
###     WeatherRepo.updateWeatherInfo()
### }
실제로 이 방법으로도 정상적으로 동작하는 것을 확인했습니다.
하지만 테스트 과정에서 Content()가 위젯 크기에 따라 여러 번 Composition되는 상황을 확인했습니다.
예를 들어 Logcat에서는 다음과 같이 서로 다른 크기에 대해 Composition이 발생했습니다.
### Content composition ... size=260.0.dp x 200.0.dp
### Content composition ... size=184.0.dp x 184.0.dp
이 경우 LaunchedEffect를 데이터 로딩 트리거로 사용하면 Composition lifecycle에 데이터 로딩 책임이 묶이게 됩니다.
반면 provideGlance()는 공식적으로 **Composable을 제공하기 전에 필요한 데이터를 로드하는 위치**로 정의되어 있습니다.
따라서 이번 문제에서는 UI Composition 내부에서 데이터를 가져오기보다, provideGlance()에서 필요한 데이터를 먼저 준비한 뒤 provideContent()로 콘텐츠를 제공하는 구조가 더 적절하다고 판단했습니다.

# 중복 호출은 괜찮을까?
여기서 또 하나 확인해야 할 문제가 있었습니다.
여러 위젯 인스턴스가 동시에 존재한다면 여러 번 provideGlance()가 호출될 수 있습니다.
그렇다면 모든 위젯에서 WeatherRepo.updateWeatherInfo()를 호출하는 것이 문제가 되지 않을까요?
다행히 기존 WeatherRepo에는 이미 중복 요청을 제한하는 로직이 존재했습니다.
### mutex.withLock(lastRun) {
###     if (lastRun.plusSeconds(TIMEOUT).isAfter(Instant.now())) {
###         return
###     } else {
###         lastRun = Instant.now()
###     }
### }
즉 호출부에서는 단순히 데이터 갱신을 요청하고, 실제로 새로운 데이터를 가져올 필요가 있는지는 Repository가 판단합니다.
실제 테스트에서도 다음과 같은 로그를 확인했습니다.
### WeatherRepo: updateWeatherInfo called with delay=2000
### WeatherRepo: updateWeatherInfo - setting state to Loading

### WeatherRepo: updateWeatherInfo called with delay=2000
### WeatherRepo: updateWeatherInfo - throttled (too soon)

### WeatherRepo: updateWeatherInfo - setting state to Available
여러 위젯 인스턴스나 Composition에서 갱신 요청이 발생하더라도 mutex와 lastRun/TIMEOUT에 의해 중복 실행이 제한되는 것을 확인했습니다.
따라서 호출부인 provideGlance()는 "갱신이 필요한가?"를 판단하지 않고 요청만 전달하고, 데이터 갱신 여부는 Repository가 담당하도록 유지할 수 있었습니다.

# 검증
수정 후 다음과 같은 상황에서 반복적으로 확인했습니다.
* 일반적인 위젯 추가
* 앱 프로세스 재시작
* Run Configuration 변경 후 재실행
* 기존 위젯이 존재하는 상태에서 재실행
* 여러 위젯 인스턴스 추가
* 서로 다른 위젯 크기에서 렌더링
* 리사이즈 없이 재실행

⠀수정 전에는 프로세스 재시작 이후 다음과 같이 Loading 상태에 머물렀습니다.
### onUpdate for 1 widgets
### provideGlance
### Rendering Loading
### Rendering Loading
수정 후에는 다음과 같이 데이터 갱신부터 콘텐츠 제공까지 정상적으로 이어졌습니다.
### onUpdate for 1 widgets
### provideGlance

### WeatherRepo: updateWeatherInfo called
### WeatherRepo: updateWeatherInfo - setting state to Loading
### WeatherRepo: updateWeatherInfo - delaying for 1000 ms
### WeatherRepo: updateWeatherInfo - setting state to Available

### Rendering Available
여러 위젯을 동시에 추가한 경우에도 모든 위젯에서 Available 상태가 정상적으로 렌더링되는 것을 확인했습니다.

# PR 제출
문제를 해결한 뒤 android/platform-samples 저장소에 PR을 제출했습니다.
* Issue: ~[\#402 — WeatherGlanceWidget not reloading after relaunch](https://github.com/android/platform-samples/issues/402)~
* Pull Request: ~[\#431](https://github.com/android/platform-samples/pull/431)~

⠀이번 PR을 통해 단순히 코드 한 줄을 추가하는 것에서 끝나는 것이 아니라, 실제 오픈소스 프로젝트의 코드와 AndroidX 내부 구현을 직접 추적하고 공식 문서를 근거로 수정 방향을 결정하는 경험을 할 수 있었습니다.

# 마무리
이번 기여를 통해 몇 가지를 배울 수 있었습니다.
### 1\. Lifecycle callback은 이름만 보고 판단하면 안 된다
### onEnabled()라는 이름만 보면 위젯이 활성화될 때마다 호출될 것처럼 느껴질 수 있습니다.
하지만 실제로는 **첫 번째 위젯 인스턴스가 생성될 때 호출되는 callback**입니다.
따라서 lifecycle callback을 사용할 때는 "언제 호출될 것 같은가"가 아니라 **정확히 어떤 조건에서 호출되는가**를 확인해야 합니다.
### 2\. 동작하는 수정과 적절한 수정은 다를 수 있다
### onUpdate()에 updateWeatherInfo()를 추가하는 것만으로도 문제는 해결할 수 있었습니다.
하지만 단순히 증상을 없애는 것과 해당 프레임워크가 의도한 구조에 맞게 문제를 해결하는 것은 다른 문제였습니다.
이번에는 onUpdate()에서 provideGlance()까지 내부 호출 흐름을 따라가고 공식 KDoc을 확인한 결과, 데이터 로딩 책임을 provideGlance()로 이동하는 것이 더 적절하다고 판단했습니다.
### 3\. 공식 문서는 중요한 설계 근거가 된다
이번 문제를 해결하는 결정적인 단서는 provideGlance()의 KDoc이었습니다.
"This is a good place to load any data needed to render the Composable."
오픈소스 코드에서 특정 위치에 로직을 추가할 때 단순히 "여기서 동작한다"는 이유만으로 결정하기보다, **해당 API가 어떤 용도로 설계되었는지를 공식 문서와 구현을 통해 확인하는 과정**이 중요하다는 것을 배웠습니다.
### 4\. 문제 해결 과정 자체가 중요한 경험이었다
처음에는 단순히 위젯이 로딩에서 멈추는 버그라고 생각했습니다.
하지만 실제로는
### 버그 재현
###     ↓
### Logcat 비교
###     ↓
### 호출되지 않은 lifecycle callback 발견
###     ↓
### AndroidX Glance 소스 추적
###     ↓
### 공식 문서 확인
###     ↓
### 수정 위치 결정
###     ↓
### 여러 환경에서 검증
###     ↓
### 오픈소스 PR 제출
이라는 과정을 거치게 되었습니다.
특히 다른 사람이 작성한 코드베이스에서 문제를 발견하고, 프레임워크 내부 동작까지 추적하면서 스스로 수정 방향을 결정해본 경험이라는 점에서 의미가 있었습니다.
이번 기여를 계기로 앞으로도 단순히 "동작하는 코드"를 작성하는 것을 넘어, **왜 이 위치에 이 코드가 있어야 하는지 설명할 수 있는 개발자**가 되는 것을 목표로 하고 있습니다.
