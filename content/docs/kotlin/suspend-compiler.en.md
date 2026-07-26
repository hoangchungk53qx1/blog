---
title: "Suspend Functions: The Heart of Coroutines"
description: "What the Suspend Function Compiler actually does"
icon: "article"
date: "2025-08-15T00:27:57+01:00"
lastmod: "2025-08-15T00:27:57+01:00"
draft: false
toc: true
weight: 999
---

# Author : ChungHA (RxMobileTeam)

# Suspend Functions: The Heart of Coroutines

For a mobile developer in general, and an Android developer in particular, RxJava, RxKotlin, RxAndroid — the whole Rx style and reactive world — used to feel like something amazing for handling asynchronous tasks. At least that's how I felt about it back in the day.

Ever since coroutines came along, Android developers have been reaching for them more and more, simply because they're easier to use than Rx — and let's be honest, Rx gets pretty hard to swallow once you go deep.

Coroutines are already pretty easy to use, and you probably already know their benefits. So in this article I won't talk about how to use them or why they're great. Instead, I want to dig deeper into the suspend function itself.

## What is a Suspend Function?

You could say that the suspend function is the center of the Coroutine universe.

Unlike a regular function, a suspend function doesn't block whatever thread it's running on. In plain terms, it's a function that can pause and resume — and cancel — and it's completely non-blocking...

Why does it have this amazing ability? Let's find out together.

### Callback Style vs Suspend Style

Let's start with the Callback style, something like this. Here I'm only demoing the happy case — everything succeeds, no errors.

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

Using callbacks is fine — this is only a few lines. But the moment you have dozens of functions nested inside each other ==> you get callback hell (you can search Google to learn more about it). Over in JS land they came up with Promises to solve this...

So what happens if we convert this to suspend style? It ends up looking like this. As you already know, a suspend function can only be called from within another suspend function, or from inside a coroutine builder.

```kotlin
suspend fun loginUser(userId: String, password: String): User { <--- suspending
  val user = userRemoteDataSource.logUserIn(userId, password) // Also a suspend function <--- suspending
  val userDb = userLocalDataSource.logUserIn(user) // Also a suspend function  <--- suspending
  return userDb
}
```

### Demo of the Related Functions

The other two functions from UserRemoteDataSource and UserLocalDataSource, I'll demo them like this:

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

Now it looks genuinely great — asynchronous code that reads like synchronous code :v ✌️

## What Does the Kotlin Compiler Do with Suspend Functions?

So what exactly does suspend do to make it so nice? For those of you who are too lazy to read the whole thing:

**TL;DR**: The Kotlin compiler takes your suspend functions and turns them into callbacks, optimized using a finite state machine. We just write plain suspend-style code, and the compiler does all that transformation for us.

Let's go a bit deeper to understand what the compiler is actually doing.

### Continuation

The first concept we'll cover is the Continuation.

Suspend functions now get transformed into something like this. I've written the transformed code back in Kotlin to make it a bit easier to follow.

Here's the code from userRemoteDataSource
```kotlin
fun logUserIn(userId: String, password: String, continuation: Continuation<*>): Any
```

Here's the code from userLocalDataSource
```kotlin
fun logUserIn(user: User, continuation: Continuation<*>): Any
```

You can already see what the compiler did: first, it changed the return type to Any, and second, it added a continuation parameter.

#### Why Return Any?

Question 1: Why does it return Any, when clearly I return a User in the remote case and a User in the local case too?

Let's step back a bit. Remember that a suspend function has the ability to suspend (pause). So when it pauses, how is it supposed to tell the caller that it's suspending? It doesn't even have a result yet, so how could it return the type you actually expected? ✌️

This is exactly the reason for returning Any or Any? — because Any is an object and can hold a token (also called a tag) named COROUTINE_SUSPENDED, which marks that the function is currently suspending. (Honestly I don't find this part super elegant; returning a union type would feel nicer — maybe in the future...)

#### What is a Continuation?

Question 2: What's the added `continuation: Continuation<?>` parameter for? If you click into it a bit, you'll see this: Link here https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
} 
```

If you dig into Continuation a bit more, you'll find it also relates to suspend points and cancellation. But within the scope of this article, I'll only talk about suspending.

We can see that the Continuation interface is a generic type that takes T as a parameter:

1. `context: CoroutineContext` is exactly the CoroutineContext, and it's the environment in which the coroutine runs (explaining this fully would add another whole section). In a nutshell, it's an Indexed Set, where a CoroutineContext.Element is itself also a CoroutineContext.

2. The `resumeWith` method is called by the coroutine when there's no longer a COROUTINE_SUSPENDED token — meaning it's no longer suspended (paused). It takes the result that got computed somehow and calls back through the Continuation, and this is the value the coroutine will produce once resumeWith completes. In short, resumeWith lets you continue (resume) from the point where the suspend function was paused (suspended/paused). It also handles exceptions via resumeWithException — you can look into that as well.

To picture it more easily, it looks like this:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any>) {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resumeWith(userDb)
}
```

Now you can see what the compiler does for us, right? It's a callback too, isn't it — well, not exactly.

## Finite State Machine

As I mentioned at the beginning, it optimizes things using a finite state machine.

Let's look at how Kotlin bytecode optimizes it:

Here's my original code:

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

When you decompile it to Java to read it, you get a file like this. It's really long, so I'll only copy the main function and the logUserIn function from userRemoteDataSource. You can decompile it yourself to see the full file.

```kotlin
---- This is main ----
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

--- This is loginUser in userRemoteDataSource`
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

Looking at these two pieces of Java code, you can see they match exactly the theory I laid out and the Kotlin compiler behavior I demoed, right?

You can see there really is an added Continuation var2 parameter, and it returns Object — the same as the Any I described above in Kotlin. So you can see the theory about Continuation and COROUTINE_SUSPENDED holds up, right?

### State Machine Code

Let me rewrite it a bit to make it easier to read:

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

This is just a condensed version. You can see that this snippet uses `when` to check the labels — so where does the label come from?

Here's how the label works. The entire snippet above gets converted into finite-state-machine style. It generates a `LoginUserStateMachine` class based on three main components, which are also the core ideas of a finite state machine:

1. **States** - These are the possible states the system can be in. For example: "State 1", "State 2", "State 3", and so on.

2. **Events** - These are events that happen in the system and can trigger a state transition. For example: "Event A", "Event B", "Event C", and other events. Each event is usually associated with an action or a condition occurring in the system.

3. **Transitions** - These are the rules that determine how the system moves from one state to another based on events and conditions.

You can read more about finite state machines here: https://en.wikipedia.org/wiki/Finite-state_machine

And now our code looks like this:

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
    
    // this function calls loginUser again to drive the
    // state machine (label will already be in the next state) and
    // result will be the computed result of the previous state
    override fun invokeSuspend(result: Any) {
      this.result = result
      loginUser(null, null, this)
    }
  }
 ...
}
```

And the place where it's called looks like this:

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
            // This is where, once the continuation is invoked, it transitions the state to label = 1, and once it becomes 1 it falls through below to do the next bit of logic — that's the next state
            continuation.label = 1
            // here the continuation gets passed along into logUserIn so it can resume
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
           // grab the result from the previous state
            continuation.user = continuation.result as User
            // it assigns label = 2 if there's still another state, and again passes the continuation object down to keep resuming
            continuation.label = 2
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Similarly, grab the result from the previous state
            continuation.userDb = continuation.result as User
            // No more labels left, so just resume with the continuation
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(...)
    }
}
```

To sum up: with a finite state machine, the Kotlin compiler transforms each suspend function into a state machine. It creates a `LoginUserStateMachine` instance and stores the continuation as a parameter, so it can pass it down to the next state — and if it's the final state, it calls resumeWith using that value.

## The Non-blocking Nature of Suspend Functions

Next, let's say a bit more about the non-blocking aspect of suspend functions.

One of the biggest design challenges of Kotlin coroutines was how to make something that can both suspend and resume, while also being non-blocking. I've already explained the suspending and resuming parts above. As for non-blocking, I noticed that both Kotlin/JVM and Kotlin/JS manage to do it, so there must be a common approach to the implementation.

The solution here is this:

According to the Kotlin authors, and just based on the principle: if you can't do the work right now, then free up the thread (here, the coroutine) inside the function — just return immediately — and then later, we can call that function again and jump straight back to the current position of the result.

As I described in part 1, this matches the suspend theory exactly 🙏

Suspend returns that object (Any). When a suspend function gets suspended, it returns the COROUTINE_SUSPENDED object to signal to its caller that it's suspending, so the caller can return (once it returns, there's nothing left blocking the thread, right?), and then later it just jumps back in.

For example: say we have suspend A(), suspend B(), and suspend C(), called in the order A -> B -> C

- C gets suspended, so it stores its state in the continuation and returns the COROUTINE_SUSPENDED object
- Now, to actually free the thread, obviously it's not just C that returns — the entire caller chain, in other words the whole stack, has to return too. The flow is: B verifies whether C returned COROUTINE_SUSPENDED — that's exactly why every suspend function has that `== COROUTINE_SUSPENDED` check. If C returned COROUTINE_SUSPENDED, then B returns too, and A returns a similar value as well. At this point the continuation is the thing holding all the state of that entire stack, and now the thread is free. There's nothing left blocking — everything has returned.

- When it's time to resume — say C had a 1-second delay — then `cont.resume` gets invoked. We call C (without needing to call the entire function chain again) by passing the continuation as a parameter, as I described.

- C reads the continuation and continues executing with those local values...

- After C returns, the same thing happens with B and then with A. Each of them has its state stored by the continuation.

## Conclusion

That's everything I understand and know about suspend functions. Thanks for reading!
