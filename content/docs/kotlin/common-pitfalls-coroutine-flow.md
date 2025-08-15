---
weight: 999
title: "Common Pitfalls Coroutine Flow"
description: ""
icon: "article"
date: "2025-08-15T14:42:02+07:00"
lastmod: "2025-08-15T14:42:02+07:00"
draft: true
toc: true
---

### **Xin chào mọi người, hôm nay mình xin nói về 1 số lỗi khi chúng ta làm việc với Kotlin CoroutineFlow**

### 1: Sử dụng sai vòng đời, gây lãng phí resource và crash app

Trong ứng dụng Android, các Flow thường được collect để hiển thị thông tin cập nhật dữ liệu trên màn hình. Tuy nhiên, bạn muốn thu thập các luồng này để đảm bảo rằng bạn không làm nhiều việc hơn mức cần thiết, gây lãng phí tài nguyên (cả CPU và Memory) hoặc leak memory chuyển sang background.

Trong Android có 2 API là `Lifecycle.repeatOnLifecycle`, và `Flow.flowWithLifecycle` giúp chúng ta quản lý resources tốt hơn

Nếu chúng ta sử dụng Flow<T> cho bất kể layer nào theo clean arch cũng được, không vấn đề gì, nhưng chúng ta nên collect data 1 cách an toàn, đây là với flow thôi nhé, còn ví dụ livedata thì khác, vì livedata khi observer theo viewlifecycleOwner thì nó observer từ onStart tới onStop rồi, nên không cần quan tâm tới wasting resouce. 

Do Flow<T> với các API hiện tại như launchIn, collect đều collect ngay cả khi background, hoặc các bạn có thể tạo ra job rồi cancel bằng tay, việc này khá thủ công và khó kiểm soát.

Đây là vòng đời 

![](https://images.viblo.asia/59a2f18b-6cde-47c8-bfb0-d03d83be9d7e.png)

Ví dụ :

```html
// Đang sử dụng callbackflow, thực chất là channel để update realtime location
fun FusedLocationProviderClient.locationFlow() = callbackFlow<Location> {
    val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult?) {
            result ?: return
            try { offer(result.lastLocation) } catch(e: Exception) {}
        }
    }
    requestLocationUpdates(createLocationRequest(), callback, Looper.getMainLooper())
        .addOnFailureListener { e ->
            close(e) // in case of exception, close the Flow
        }
    // clean up when Flow collection ends
    awaitClose {
        removeLocationUpdates(callback)
    }
}
```

Và chúng ta sử dụng collect update lên view như này :

```html
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Collects từ View khi mà vòng đời đạt tới STARTED
        // SUSPENDS khi vòng đời bị STOPPED.
        // Tất cả collector cancel khi mà vòng đời DESTROYED.
        lifecycleScope.launchWhenStarted {
            locationProvider.locationFlow().collect {
               ....
            } 
        }
        // Lỗi tương tự khi sử dụng
        // - lifecycleScope.launch { /* Collect from locationFlow() here */ }
        // - locationProvider.locationFlow().onEach { /* ... */ }.launchIn(lifecycleScope)
    }
}
```

Lỗi ở đây là gì `lifecycleScope.launchWhenStarted` suspend việc thực thi coroutine. Các location mới không được xử lý, mà khi đó  producer `callbackFlow` vẫn gửi các location. Việc sử dụng các API `lifecycleScope.launch` hoặc `launcherIn` thậm chí còn nguy hiểm hơn vì view tiếp tục sử dụng các location (lỡ may ref tới đâu đó trong view mà lúc đó binding đã chết, (chết tiến trình...)) ngay cả khi nó ở trong background! --> Dẫn tới crash App

Để giải quyết vấn đề này chúng ta có thể cancel thủ công như này

```html
class LocationActivity : AppCompatActivity() {

    // Tạo ra job
    private var locationUpdatesJob: Job? = null

    override fun onStart() {
        super.onStart()
        locationUpdatesJob = lifecycleScope.launch {
            locationProvider.locationFlow().collect {
                ....
            } 
        }
    }

    override fun onStop() {
        // Dừng việc lắng nghe ở onStop
        locationUpdatesJob?.cancel()
        super.onStop()
    }
}
```

Đây cũng là cách, nhưng lại tạo ra nhiều code hơn, lỡ lắng nghe chục cái flow thì..., rất khó quản lý.

Thay vào đó, như mình giới thiệu ở vòng đời bên trên, có **Lifecycle.repeatOnLifecycle**

```html
lass LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Ở đây tạo ra 1 coroutine mới, vì repeatOnLifecycle là suspend function, tránh việc suspending lẫn nhau
        lifecycleScope.launch {
          // Ở đây truyền repeatOnLifecycle vào, thực thi trên vòng đời viewLifeCycleOwner,
          // Nó sẽ được thực thi khi vòng đời đạt tới STARTED và cancel khi STOPPED
          // Hơn nữa, nó sẽ tự động restart khi vòng đời lại STARTED lần nữa, vì vậy nó mới gọi là repeatOnLifecycle 
            lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // An toàn khi update Location từ  STARTED
                // Và dừng việc collect khi STOPPED
                locationProvider.locationFlow().collect {
                    
                }
            }
        }
    }
}
```

Trên đây là cách collect an toàn từ layerUI, đây là xml, còn trong compose mới có API gần đây collectAsStateWithLifecycle, các bạn có thể tham khảo.

### 2: Sử dụng emit - **tryEmit** - update mà không hiểu rõ cách implement.

Khi sử dụng **MutableStateFlow** thì chắc chắn bạn sẽ cần update value, tuy nhiên có nhiều hàm update như `emit()/tryEmit()/update()` . Vậy sử dụng thằng nào khi nào ?

* **suspend fun emit()**: Dùng khi muốn emit một value nào đó, hàm sẽ bị suspend nếu flow được cài đặt `onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND` và số lượng phần tử buffer vượt quá `extraBufferCapacity`. Chỉ đến khi các value được collect thì các emit function mới tiếp tục được thực thi.

* **fun tryEmit()**: Hàm emit value cho flow mà không làm suspend, nếu việc emit thành công thì kết quả return true. Tuy nhiên, nếu flow được cài đặt `onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND` và số lượng phần tử buffer vượt quá `extraBufferCapacity` thì kết quả sẽ return false.

* **fun update()**: Dùng khi update value của `MutableStateFlow`, các collector sẽ chỉ nhận được event change khi **currentValue != newValue**. Hàm này sẽ tạo một vòng lặp vô hạn để so sánh current value với new value, vòng lặp sẽ được ngắt chỉ newValue == oldValue. Nói kĩ hơn một chút về hàm này, nếu class của value là **data class** thì mặc định sẽ so sánh tất cả các field xem có equals với nhau hay không, nếu không thì so sánh địa chỉ của 2 object (hàm equals của class). Vậy nên ở đây có một số lưu ý cần phải nhớ:

    * Tuyệt đối không return false khi override hàm equals(): Điều này sẽ làm thread bị blocking forever.

    * Nếu có nhiều thread cùng truy cập và update StateFlow đồng thời thì hàm update sẽ lock thread hiện tại cho đến khi newValue được update thành công.

* Ví dụ:

###

![](https://storage.googleapis.com/iwiki/1711702402441_blobid0.png)

Ở đây mình tạo activity demo việc nhấn button và update UI lên trên màn hình thôi

###

![](https://storage.googleapis.com/iwiki/1711702402448_blobid1.png)

Cách 1: .value để update. các bạn thấy mình dùng ArrayList (not stability) , và kết quả là dù nhấn button thì giá trị cũng không được cập nhập lên trên màn hình do data class sẽ mặc định so sánh các field chứ không so sánh địa chỉ => Object mới có địa chỉ khác nhưng content giống  => .value nhưng collector không nhận thay đổi.

![](https://storage.googleapis.com/iwiki/1711702402453_blobid2.png)

Cách 2: Dùng update, cũng failed , thậm chí chỗ cách 2 , mình còn clone hẳn ra cái reference mới, mà dữ liệu không update lên màn hình, cũng do data class hết, như mình vừa trình bày bên trên

![](https://storage.googleapis.com/iwiki/1711702402459_blobid3.png)

Cách sửa : Sử dụng data class và các immutable field là cách tốt nhất để tránh bị lỗi này, code càng immutable càng tốt, càng ít bug.

### 3: Sử dụng khi refer tới value của StateFlow

Chúng ta lấy value như thế này

```kotlin
private val topicStateFlow = MutableStateFlow(Topic())
val topicCurrent = topicStateFlow.value
```

Thấy không vấn đề gì đúng không, thực chất MutableStateFlow là interface. 

```kotlin
public interface MutableStateFlow<T> : StateFlow<T>, MutableSharedFlow<T> {
    public override var value: T
}
```

Thấy rằng `value` là một generic variable, khi StateFlow được update giá trị mới thì giá trị của nó sẽ được thay đổi theo. Như vậy thì thì việc gán `val topicCurrent = topicStateFlow.value oke mà?`

```kotlin
Thực chất bên dưới là nếu chỉ nhìn qua interface thì ta sẽ nghĩ rằng value 
là một variable nên ta có thể tạo một biến để refer vào lấy giá trị khi cần. 
Tuy nhiên ở implementation thì thực chất nó là một backing field,value thực sự được hold bởi _state. 
Đặc điểm của backing field là một function được ẩn dưới getter/setter của một variable

private class StateFlowImpl<T>(
    initialState: Any // T | NULL
) : AbstractSharedFlow<StateFlowSlot>(), MutableStateFlow<T>, CancellableFlow<T>, FusibleFlow<T> {
    private val _state = atomic(initialState) // T | NULL
    private var sequence = 0 // serializes updates, value update is in process when sequence is odd

    @Suppress("UNCHECKED_CAST")
    public override var value: T
        get() = NULL.unbox(_state.value)
        set(value) { updateState(null, value ?: NULL) }
}

thì topicCurrent sẽ chỉ được set 1 lần thông qua backing field của value.
Vậy nên khi StateFlow update giá trị cho biến _state thì topicCurrent vẫn sẽ hold giá trị cũ.
```

Cách để không bị lỗi này 

1. Tạo một variable StateFlow từ MutableStateFlow thông qua hàm asStateFlow() để lấy `value` khi cần.

2. Nếu sử dụng biến để hold current value của StateFlow thì cần khai báo nó ở dạng backing field.

```kotlin
private val topicStateFlow = MutableStateFlow(Topic())
    
    val currentTopic
        get() = topicStateFlow.value
```

### 4 : SharedFlow không nên sử dụng Single Event

StateFlow là child class của SharedFlow, là state holder vì vậy nó luôn hold trong mình một `value` tương tự như LiveData. Khi dùng LiveData thì hẳn các bạn đã khắc phục vấn đề data bị replay bằng cách sử dụng 1 observer và dùng atomic để không cho thằng nào vào observer nữa đúng không. (Value sẽ chỉ được collect 1 lần và không replay lại cho new subscriber).

SharedFlow cho phép tuỳ biến `replay` và `extraBufferCapacity`, có khi nào bạn sử dụng **MutableSharedFlow(replay = 0, extraBufferCapacity = 0)** để sử dụng SharedFlow như một SingleLiveEvent chưa ? Test qua thì cũng ổn áp đấy, event sẽ không bị replay cho collector mới.

**NHƯNG**..., nếu các collector stop việc collect và SharedFlow vẫn tiếp tục được emit thì sẽ dẫn đến lỗi bị miss event (Do cả replay và extraBufferCapacity đều được set bằng 0). 

Miss khi nào, như vòng đời bên trên mình trình bày, flow có thể bị suspend,vòng đời vào onStoped... với ví dụ sau

Ví dụ: Sử dụng **MutableSharedFlow(replay = 0, extraBufferCapacity = 0)** để share error khi call API gặp lỗi. Ở phía UI collect errorFlow khi app foreground, khi background UI ngừng collect data nhưng background vẫn call API thì gặp lỗi và emit lỗi vào errorFlow. Khi UI trở lại foreground thì sẽ không được hiển thị lỗi kết nối => Miss event.

Vậy giải pháp ở đây là gì ? Chúng ta cần một flow đáp ứng được yêu cầu sau:

* Event phải được consume và consume duy nhất 1 lần.

* Event phải persist và không bị miss kể cả không có collector nào.

Câu trả lời chính là **Channel**  chúng ta sẽ có hotFlow cho phép emit event và collect event duy nhất một lần như một Single Event.

### 4 : Channel không sử dụng đúng Dispatcher

Channel là cách giao tiếp như 1 blocking queue, nhưng từ kotlin 1.4 Channel có prompt cancellation guarantee of Channel nên phải collect và send event trong main.immediate, nếu không đảm bảo sẽ bị mất event (undelivered). 

Sử dụng mainThread vẫn sai, nó sẽ khôgn excute ngay, do đang xử lý 1 handler callback hoặc choreographer animation frame stage

Do đó , mainthread vẫn có độ trễ là như vậy, vì vậy hãy dùng Dispatchers.Main.immediate, trySend nonblocking, để send, collect event\
\
Trên đây là toàn bộ những lỗi và kinh nghiệm của mình khi sử dụng coroutineflow sai do không hiểu bản chất mà dễ gặp phải, được mình đúc rút từ nhiều nguồn, và cả kinh nghiệm bản thân, Cảm ơn các bạn đã theo dõi.
