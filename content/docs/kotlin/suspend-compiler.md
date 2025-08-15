---
title: "Suspend Function trung tâm trong Coroutine"
description: "Suspend Function Compiler làm gì"
icon: "article"
date: "2025-08-15T00:27:57+01:00"
lastmod: "2025-08-15T00:27:57+01:00"
draft: false
toc: true
weight: 999
---

# Tác giả : ChungHA (RxMobileTeam)

# Suspend Function trung tâm trong Coroutine

Đối với 1 lập trình viên Mobile nói chung, và Android nói riêng, trước đây đối với mình, RxJava, RxKotlin, RxAndroid nói chung Rx style và reactive là thứ gì đó tuyệt vời để asynchonous task

Từ khi coroutines ra đời, các lập trình viên Android chuyển sang dùng nhiều hơn vì nó dễ dùng hơn Rx, vì Rx dùng sâu khá khó nuốt.

Đối với coroutine thì các bạn khá dễ dùng rồi, cũng như lợi ích của nó rồi, trong bài viết này mình sẽ không nói tới cách dùng và lợi ích của nó nữa mà phân tích kỹ hơn về vấn đề suspend function

## Suspend Function là gì?

Có thể nói suspend function là trung tâm trong vũ trụ Coroutine.

Không giống như function thường, suspend function không block bất kì thread nào mà nó chạy trên đó, nói thuần tiếng việt nó là function có khả năng tạm dừng và tiếp tục,và huỷ và nó hoàn toàn non-blocking...

Tại sao nó có khả năng tuyệt vời đó, mời các bạn cùng mình tìm hiểu nhé.

### Callback Style vs Suspend Style

Từ đầu chúng ta sẽ đi với style code Callback dạng như thế này, Ở đây mình chỉ demo tới việc happy case, success hết nhé

```kotlin
fun loginUser(userId: String, password: String, userResult: Callback<User>) {
  userRemoteDataSource.logUserIn { user ->
    // Successful network request
    userLocalDataSource.logUserIn(user) { userDb ->
      // Result saved in DB
      userResult.success(userDb)
    }
  }
}
```

Dùng callback thì cũng oke, đây chỉ có vài dòng, nhưng lỡ có hàng chục function nested nhau ==> dẫn tới callback hell, (các bạn có thể search google để tìm hiểu hơn), bên js mới sinh ra Promise để giải quyết...

Với cái này chuyển qua style suspend thì sao, nó sẽ trông như thế này, các bạn đã biết thì suspend chỉ gọi được trong 1 suspend func, hoặc trong 1 coroutine builder

```kotlin
suspend fun loginUser(userId: String, password: String): User { <--- suspending
  val user = userRemoteDataSource.logUserIn(userId, password) // Cũng là suspend function nhé <--- suspending
  val userDb = userLocalDataSource.logUserIn(user) // Cũng là suspend function nhé  <--- suspending
  return userDb
}
```

### Demo các hàm liên quan

2 hàm còn lại từ UserRemoteDataSource và UserLocalDataSource, mình sẽ demo như thế này:

```kotlin
// userRemoteDataSource
suspend fun logUserIn(userId: String, password: String): User {
        delay(1000) <-- suspending
        return User(userId,password)
}
```

```kotlin
// userLocalDataSource.kt
suspend fun logUserIn(user: User): User {
        delay(1000) <-- suspending
        return user
}
```

Giờ nhìn nó tuyệt vời thật, code asynchonous mà như style synchonous :v ✌️

## Kotlin Compiler làm gì với Suspend Function?

Vậy suspend nó làm gì mà hay quá vậy, thì cho bạn nào lười đọc thì

**Tóm tắt**: Kotlin compiler sẽ sử dụng các suspend function và chuyển đổi chúng thành callback và được tối ưu hóa bằng cách sử dụng finite state machine, và chúng ta chỉ viết style suspend bình thường còn chuyển hoá đó do Kotlin complier làm cho chúng ta

Đi sâu hơn 1 chút để hiểu complier nó làm gì nhé.

### Continuation

Chúng ta tới với concept đầu tiên là Continuation

Các suspend fuction giờ nó biến thành dạng như này, code này mình mô tả sang kotlin cho các bạn dễ nhìn chút nhé.

Đây là code từ userRemoteDataSource
```kotlin
fun logUserIn(userId: String, password: String, continuation: Continuation<*>): Any
```

Đây là code từ userLocalDataSource
```kotlin
fun logUserIn(user: User, continuation: Continuation<*>): Any
```

Các bạn có thể thấy Complier nó đã làm gì, thứ nhất nó dữ liệu return là Any, thứ hai thêm param continuation

#### Tại sao return Any?

Câu hỏi thứ 1: là tại sao nó return Any, trong khi rõ ràng mình return chỗ remote là User, còn local là User luôn mà.

Quay lại 1 chút thì hàm suspend như các bạn còn nhớ, nó có chứng năng suspend (tạm dừng), vậy lúc nào tạm dừng nó sao mà thông báo cho caller biết nó đang suspending được và có kết quả đâu mà return cái type mong muốn ✌️

Đây chính vấn đề return Any or Any?, bởi vì Any là object và có thể chứa được 1 token hay còn gọi là tag là COROUTINE_SUSPENDED nhằm cho việc đánh dấu function đó đang bị suspending. (Mình thấy đoạn này cũng chưa hay, return union type thấy hay hơn, trong tương lai biết đâu ...)

#### Continuation là gì?

Câu hỏi thứ 2: Tham số continuation: Continuation<?> thêm vào làm gì, click vào 1 chút ta sẽ thấy: Link ở đây https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
} 
```

Cái Continuation các bạn ngâm cứu thêm sẽ thấy nó còn liên quan tới suspend point, cancellation nữa cơ, mà phạm vi bài này mình chỉ nói tới suspend thôi.

Chúng ta thấy Continuation interface là generic type nhận T làm tham số:

1. với context: CoroutineContext chính là CoroutineContext và chính là enviroment cho coroutine thực hiện (đoạn này nói nó lại dài thêm 1 đoạn nữa), sơ sơ thì nó là Indexed Set, CoroutineContext.Element cũng chính là CoroutineContext

2. method resumeWith được thằng coroutine gọi khi mà không còn cái token COROUTINE_SUSPENDED, nghĩa là không bị suspend (tạm dừng nữa), nó sẽ lấy kết quả được tính toán như thế nào đó và callback lại thông qua Continuation, và đây là giá trị mà coroutine sẽ tạo ra khi resumeWith hoàn thành. Tóm lại resumeWith nó cho phép tiếp tục (resume) từ nơi mà suspend function bị tạm dừng (suspend/paused) Ngoài ra nó còn handle Exception với resumeWithException nữa, các bạn tìm hiểu thêm nhé.

Hình dung dễ hơn thì nó sẽ như thế này:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any>) {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resumeWith(userDb)
}
```

Chúng ta thấy complier nó làm gì cho chúng ta rồi chứ, nó cũng là callback đúng không, không hẳn thế.

## Finite State Machine

Như mình trình bày ban đầu thì nó tối ưu hóa bằng cách sử dụng finite state machine.

Cùng xem kotlin bytecode nó tối ưu hoá kiểu gì nhé:

Đây là code ban đầu của mình:

```kotlin 
data class User(val userId: String, val password: String)
```

```kotlin
fun main(): Unit = runBlocking {
    val result = loginUser("ChungHA", "123456a@A")
    println(result)
}
```

```kotlin
suspend fun loginUser(userId: String, password: String): User {
    val user = logUserIn(userId, password)
    val userDb = logUserIn(user)
    return userDb
}
```

```kotlin
// userRemoteDataSource.kt
suspend fun logUserIn(userId: String, password: String): User {
    delay(1000)
    return User(userId, password)
}
```

```kotlin
// userLocalDataSource.kt
suspend fun logUserIn(user: User): User {
    delay(1000)
    return user
}
```

Khi decompile sang Java để đọc nó sẽ ra dạng file như này, dài quá nên mình chỉ copy đoạn main và hàm logUserIn ở userRemoteDataSource thôi nhé, các bạn có thể tự decompile sang để xem full file nhé.

```kotlin
---- Đây là main ----
public static final void main() {
  BuildersKt.runBlocking$default((CoroutineContext)null, (Function2)(new Function2((Continuation)null) {
     int label;

     @Nullable
     public final Object invokeSuspend(@NotNull Object $result) {
        Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        Object var10000;
        switch (this.label) {
           case 0:
              ResultKt.throwOnFailure($result);
              Continuation var10002 = (Continuation)this;
              this.label = 1;
              var10000 = Demo_suspendKt.loginUser("ChungHA", "123456a@A", var10002);
              if (var10000 == var3) {
                 return var3;
              }
              break;
           case 1:
              ResultKt.throwOnFailure($result);
              var10000 = $result;
              break;
           default:
              throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
        }

        User result = (User)var10000;
        System.out.println(result);
        return Unit.INSTANCE;
     }

     @NotNull
     public final Continuation create(@Nullable Object value, @NotNull Continuation $completion) {
        return (Continuation)(new <anonymous constructor>($completion));
     }

     @Nullable
     public final Object invoke(@NotNull CoroutineScope p1, @Nullable Continuation p2) {
        return ((<undefinedtype>)this.create(p1, p2)).invokeSuspend(Unit.INSTANCE);
     }

     // $FF: synthetic method
     // $FF: bridge method
     public Object invoke(Object p1, Object p2) {
        return this.invoke((CoroutineScope)p1, (Continuation)p2);
     }
  }), 1, (Object)null);
}

--- Đây là loginUser ở userRemoteDataSource`
@Nullable
public static final Object loginUser(@NotNull String userId, @NotNull String password, @NotNull Continuation var2) {
  Object $continuation;
  label27: {
     if (var2 instanceof <undefinedtype>) {
        $continuation = (<undefinedtype>)var2;
        if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
           ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
           break label27;
        }
     }

     $continuation = new ContinuationImpl(var2) {
        // $FF: synthetic field
        Object result;
        int label;

        @Nullable
        public final Object invokeSuspend(@NotNull Object $result) {
           this.result = $result;
           this.label |= Integer.MIN_VALUE;
           return Demo_suspendKt.loginUser((String)null, (String)null, (Continuation)this);
        }
     };
  }

  Object var10000;
  label22: {
     Object $result = ((<undefinedtype>)$continuation).result;
     Object var7 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
     switch (((<undefinedtype>)$continuation).label) {
        case 0:
           ResultKt.throwOnFailure($result);
           ((<undefinedtype>)$continuation).label = 1;
           var10000 = logUserIn(userId, password, (Continuation)$continuation);
           if (var10000 == var7) {
              return var7;
           }
           break;
        case 1:
           ResultKt.throwOnFailure($result);
           var10000 = $result;
           break;
        case 2:
           ResultKt.throwOnFailure($result);
           var10000 = $result;
           break label22;
        default:
           throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
     }

     User user = (User)var10000;
     ((<undefinedtype>)$continuation).label = 2;
     var10000 = logUserIn(user, (Continuation)$continuation);
     if (var10000 == var7) {
        return var7;
     }
  }

  User userDb = (User)var10000;
  return userDb;
}
```

2 phần code java các bạn có thể thấy đúng như lý thuyết mà mình trình bày và demo lại kotlin complier rồi đúng không?

Các bạn có thể thấy đúng là có thêm tham số Continuation var2 và trả về Object thì tương tự như Any như kotlin mình trình bày ở trên. Các bạn thấy đúng lý thuyết về Continuation và COROUTINE_SUSPENDED rồi đúng không?

### State Machine Code

Mình sẽ tiến hành viết lại 1 chút cho các bạn dễ nhìn hơn nhé:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any>) {
  when(label) {
    0 -> { // Label 0 -> first execution
        userRemoteDataSource.logUserIn(userId, password)
    }
    1 -> { // Label 1 -> resumes from userRemoteDataSource
        userLocalDataSource.logUserIn(user)
    }
    2 -> { // Label 2 -> resumes from userLocalDataSource
        completion.resume(userDb)
    }
    else -> throw IllegalStateException(...)
  }
}
```

Đây là mình tóm gọn thôi, các bạn thấy đoạn dưới này nó đã dùng when để check các label, vậy label từ đâu mà có?

Thì đây là cách mà label hoạt động, toàn bộ đoạn code trên sẽ được covert sang style finite state machine, nó sẽ tạo ra 1 class LoginUserStateMachine và dựa vào 3 thành phần chính, đây cũng chính là tư tưởng của finite state machine:

1. **Trạng thái (States)** - Là các trạng thái có thể xảy ra trong hệ thống. Ví dụ: "Trạng thái 1", "Trạng thái 2", "Trạng thái 3", và cứ tiếp tục.

2. **Sự kiện (Events)** - Là các sự kiện xảy ra trong hệ thống, có thể gây ra sự chuyển đổi trạng thái. Ví dụ: "Sự kiện A", "Sự kiện B", "Sự kiện C", và các sự kiện khác. Mỗi sự kiện thường liên quan đến một hành động hoặc điều kiện xảy ra trong hệ thống.

3. **Chuyển đổi trạng thái (Transitions)** - Là các quy tắc xác định cách mà hệ thống chuyển đổi từ một trạng thái sang trạng thái khác dựa trên sự kiện và điều kiện.

Finite state machine các bạn có thể đọc thêm Link: https://en.wikipedia.org/wiki/Finite-state_machine

Và giờ code của chúng ta sẽ như này:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any>) {
  class LoginUserStateMachine(
    // completion parameter is the callback to the function 
    // that called loginUser
    completion: Continuation<Any>
  ): CoroutineImpl(completion) {
    // Local variables of the suspend function
    var user: User? = null
    var userDb: UserDb? = null
    // Common objects for all CoroutineImpls
    var result: Any? = null
    var label: Int = 0
    
    // hàm này gọi lại loginUser
    // state machine (label sẽ ở trạng thái tiếp theo) và
    // kết quả sẽ là kết quả tính toán của trạng thái trước đó
    override fun invokeSuspend(result: Any) {
      this.result = result
      loginUser(null, null, this)
    }
  }
 ...
}
```

Và nơi được gọi nó sẽ như này:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any>) {

    class LoginUserStateMachine(
        // completion parameter is the callback to the function that called loginUser
        completion: Continuation<Any>
    ): CoroutineImpl(completion) {
        // objects to store across the suspend function
        var user: User? = null
        var userDb: UserDb? = null

        // Common objects for all CoroutineImpl
        var result: Any? = null
        var label: Int = 0

        // this function calls the loginUser again to trigger the 
        // state machine (label will be already in the next state) and 
        // result will be the result of the previous state's computation
        override fun invokeSuspend(result: Any?) {
            this.result = result
            loginUser(null, null, this)
        }
    }

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Đoạn này chính là khi continuation được gọi, nó sẽ chuyển trạng thái thành label = 1 , và khi thành 1 nó sẽ nhảy xuống dưới làm logic gì đó tiếp, gọi là next state 
            continuation.label = 1
            // đoạn này continuation sẽ được truyền tiếp vào logUserIn để resume
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
           // lấy kết quả từ trạng thái trước
            continuation.user = continuation.result as User
            // nól lại gán label = 2 nếu còn state, và lại truyền cái object continuation xuống để resume tiếp
            continuation.label = 2
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Tương tự lấy kết quả từ trạng thái trước
            continuation.userDb = continuation.result as User
            // Check ko còn label nào nữa thì resume với continuation thôi
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(...)
    }
}
```

Tóm lại là finite state machine Kotlin compiler biến đổi mỗi suspend function thành một state machine, nó sẽ tạo ra LoginUserStateMachine instance và lưu trữ continuation như 1 tham số, để có thể truyền xuống state tiếp theo và nếu là state cuối thì resumeWith với value đó.

## Non-blocking của Suspend Function

Tiếp theo, chúng ta sẽ nói thêm tới việc non-blocking của suspend function.

Một trong những thách thức lớn nhất của kotlin coroutine khi thiết kế là làm sao nó vừa suspending, vừa có thể resume, và khả năng non-blocking thì 2 cái suspeding và resume thì mình đã giải thích bên trên rồi. Còn non-blocking thì mình thấy kotlin Jvm, hay kotlin js đều làm được vậy luôn, chắc chắn phải có cách implemation chung rồi.

Giải pháp ở đây chính là:

Theo tác giả Kotlin và dựa trên nguyên lý thôi: Nếu mà không thì giải phóng 1 thread (ở đây là coroutine) bên trong 1 function, thì return luôn, rồi sau đó, chúng ta có thể gọi lại hàm đó và chuyển thẳng đến vị trí hiện tại của kết quả.

Như mình trình bày phần 1 thì đúng lý thuyết suspend luôn 🙏

Suspend nó trả về object (Any) đó, khi 1 suspend func bị suspended nó trả về đối tượng COROUTINE_SUSPENDED thông báo cho caller của nó là nó đang suspending để caller return (return rồi, còn block thread gì nữa, đúng không), rồi sau nó jump lại thôi.

Ví dụ: chúng ta có suspend A(), suspend B() và suspend C(), thứ tự gọi là A -> B -> C

- C bị suspending, nó sẽ lưu trữ state trong continuation và return ra đối tượng COROUTINE_SUSPENDED
- Giờ muốn thread được giải phóng, thì tất nhiên không chỉ C return, mà toàn bộ caller, hay nói cách khác cả stack cũng phải return luôn, quy trình nó sẽ B nó xác minh xem thằng C nó return COROUTINE_SUSPENDED không, nên mỗi suspend mới có đoạn check == COROUTINE_SUSPENDED là như vậy, nếu C nó đã return COROUTINE_SUSPENDED thì B nó cũng return, A cũng return value tương tự luôn, lúc này thằng continuation là thằng lưu trữ state toàn bộ của stack đó luôn, và bây giờ thread đã free rồi. còn gì block nữa đâu, return hết rồi.

- Khi cần resume, ví dụ C delay 1 giây, thì cons.resume sẽ được invoke, chúng ta gọi C (không cần call toàn bộ lại function) bằng cách passed cái continuation thành param như mình trình bày.

- C sẽ đọc continuation và sẽ thực thi tiếp với những giá trị local...

- Sau khi C trả về, tương tự với B và sau đó với A. Mỗi thằng đều do continuation lưu trữ state.

## Kết luận

Trên đây là toàn bộ hiểu biết, kiến thức của mình về suspend function, cảm ơn các bạn đã đọc!
