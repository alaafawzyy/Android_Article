
لما تيجي تعمل تطبيق، ساعات بتحتاج تنفذ حاجات تقيلة زي تحميل صورة من النت، وساعات تانية المستخدم بيضغط علي button  ولازم التطبيق يرد عليه بسرعة.

لو شغلت كل ده على نفس   الـ **Main Thread* التطبيق هيهنّج ومش هيشتغل كويس.  
ولو استخدمت Threads تانية، هتدخل في تعقيدات كتير زي إنك تنظم الوقت بينهم، وده بيخلي الكود صعب ومليان مشاكل.

من المشاكل دي:

- الـ app يعلّق وميفكّش (اسمه **Deadlock**).
    
- يحصل لخبطه في ترتيب الحاجات ويدي نتايج غلط (**Race Condition**).
    
- استهلاك كبير للـ resource لو بتنقل بين threads كتير (**Resource Consuming**).
    

هنا بييجي دور **Coroutines** في Kotlin.  
هي طريقة  بتخليك تنفذ أكتر من حاجة في نفس الوقت، من غير ما تدخل في دوشة تنظيم ال thread  بنفسك.  
يعني بدل ما الكود يبقى معقد، **Coroutines** بتخليه سهل ومفهوم، والابب  بيشتغل كويس 




## 1. **runBlocking**
- الـ `runBlocking` دي علشان أقدر أشغّل الكوروتين، يعني زي المكان اللي هيشتغل فيه الكوروتين، بس بيخلّي الـ **Main Thread** يستناها لحد ما تخلّص. يعني بيوقّف الدنيا لحد ما الكود جواه يخلّص.
- ودي مش أفضل حاجة لأنها بتوقّف الـ **Main Thread**.

### مثال الكود:
```kotlin
runBlocking {
    launch {
        for (i in 1..8) {
            println("waiting finger scan $i ${Thread.currentThread().name}")
            delay(1000)
        }
    }
    launch {
        for (i in 1..8) {
            println("waiting user integration $i ${Thread.currentThread().name}")
            delay(1000)
        }
    }
}
```

### إيه اللي بيحصل هنا؟
- الـ `runBlocking` بيشغّل الكود على الـ **Main Thread**.
- جواه بنستخدم `launch`، ودي زي ما تكون بتفتح كوروتين جديد يشتغل لوحده.
- الكوروتينز بتشتغل مع بعض (زي **parallel**) بس على نفس الـ **thread**.
- الـ `delay(1000)` بيوقّف الكوروتين ثانية ويسيب الـ **thread** يروح للكوروتين التاني.

### الـ Output:
```
waiting finger scan 1 main
waiting user integration 1 main
waiting finger scan 2 main
waiting user integration 2 main
...
```
- ليه متلخبط؟ عشان الـ **thread** بيتنقّل بين الكوروتينز كل ما واحد فيهم يعمل `delay`.

---

## 2. **Suspend Functions**
- الـ **Suspend Functions** دي فانكشنز مكتوب جنبها `suspend`، ودي بتعرف تتوقف وترجع تكمّل من غير ما توقّف الـ **thread**، يعني بتعمل **stop/resume**.
- زي فيلم كده **رضا** لما كانوا في سباق جري، واحد (**suspend fun**) بيجري، لو تعب يقف ويخرج، يدخل التاني، ولو التاني وقف يدخل التالت، وهكذا بيفضلوا يبدّلوا لحد النهاية.
- لازم تشغّلها جوا حاجة زي `runBlocking` أو **Coroutine Scope**.

### مثال الكود:
```kotlin
suspend fun waitFingerPrint() {
    for (i in 1..8) {
        println("waiting finger scan $i ${Thread.currentThread().name}")
        delay(1000)
    }
}

suspend fun waitUserIntegration() {
    for (i in 1..8) {
        println("waiting user integration $i ${Thread.currentThread().name}")
        delay(1000)
    }
}
```

### إيه اللي بيحصل؟
- زي المثال الأول، الكوروتينز هتشتغل مع بعض على نفس الـ **thread**، والـ **output** هيبقى زي اللي فوق.

---

## 3. **Dispatcher**

### الـ Dispatcher إيه؟
- الـ **Dispatcher** هو اللي بيختار الـ **thread** اللي الكوروتين هيشتغل عليه.
- بيقدر يبدّل الـ **threads** لو الكوروتين اتوقف (**suspend**) ورجع (**resume**)، يعني ممكن يرجع على **thread** مختلف عن اللي بدأ بيه.

### أنواع الـ Dispatchers:

1. **`Dispatchers.IO`**  
    مخصص للعمليات التقيلة اللي بتحتاج وقت، زي الـ (**Network**) أو التعامل مع الـ (**Database**). بيستخدم **Thread Pool** كبير ومتخصص للتاسكات دي.
    
2. **`Dispatchers.Main`**  
    بيشتغل على **Main Thread**، وده اللي بنستخدمه في التاسك المرتبطة  بـ (**UI**) في أندرويد، زي show message 
3. **`Dispatchers.Default`**  
    معالجة البيانات والعمليات الحسابيه  التقيلة اللي مش مرتبطة بالـ UI. بيستخدم **Thread Pool** ديناميك حسب عدد أنوية البروسيسور  (**CPU cores**).
    
4. **`Dispatchers.Unconfined`**  
    مش بيكون مرتبط بـ Thread معيّن، بيبدأ في نفس الـ Thread اللي استُدعي فيه، وبيكمل في أي Thread تاني متاح. نادرًا ما نستخدمه، وغالبًا بيكون لحالات خاصة جدًا.
    


### مثال الكود:
```kotlin
runBlocking(Dispatchers.Default) {
    launch {
        waitFingerPrint()
    }
    launch(Dispatchers.Default) {
        waitUserIntegration()
    }
}

suspend fun waitFingerPrint() {
    for (i in 1..8) {
        println("waiting finger scan $i ${Thread.currentThread().name}")
        delay(1000)
    }
}

suspend fun waitUserIntegration() {
    for (i in 1..8) {
        println("waiting user integration $i ${Thread.currentThread().name}")
        delay(1000)
    }
}
```

### إيه اللي بيحصل؟
- `runBlocking(Dispatchers.Default)` بيشغّل الكوروتينز على الـ **Default Thread Pool**.
- أول `launch` بيشغّل `waitFingerPrint()` على **thread** زي `DefaultDispatcher-worker-1`.
- تاني `launch(Dispatchers.Default)` بيشغّل `waitUserIntegration()` على **thread** تاني (أو نفس الـ **thread** حسب الـ **pool**).
- الـ `delay(1000)` بيوقّف الكوروتين ثانية، والـ **Dispatcher** ممكن يبدّل الـ **threads** لما الكوروتين يرجع (**resume**).

### الـ Output:
```
waiting finger scan 1 DefaultDispatcher-worker-1
waiting user integration 1 DefaultDispatcher-worker-2
waiting finger scan 2 DefaultDispatcher-worker-3
waiting user integration 2 DefaultDispatcher-worker-1
...
```
- التبديل ده بيحصل عشان الـ **Default Dispatcher** بيوزّع الشغل على الـ **Thread Pool**، وكل مرة الكوروتين يتوقف ويرجع، ممكن ياخد **thread** مختلف.

---

## 4. **Coroutine Scope**

### إيه هو Coroutine Scope؟
- الـ **Coroutine Scope** زي **Block** بتحط فيه الكوروتينز علشان تتحكم هتبدأ وتنتهي إمتى.
- بيخلّيك توقّف الكوروتين لو خرجت من الشاشة (زي لما بتحمّل بيانات وتسيب الـ **Activity**).

### إزاي بيشتغل؟
- الـ **Scope** ليه **Dispatcher** (يختار الـ **thread**) و**Job** (يوقّف الكوروتينز).
- لو عايز توقّف الكوروتينز، اعمل `cancel()` للـ **Scope**.

### مثال الكود:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default)
    scope.launch { waitFingerPrint() }
    scope.launch { waitUserIntegration() }
    delay(2000)
    scope.cancel() // الكوروتينز تتوقف
}

suspend fun waitFingerPrint() {
    for (i in 1..8) {
        println("finger scan $i ${Thread.currentThread().name}")
        delay(1000)
    }
}

suspend fun waitUserIntegration() {
    for (i in 1..8) {
        println("user integration $i ${Thread.currentThread().name}")
        delay(1000)
    }
}
```

### الـ Output:
```
finger scan 1 DefaultDispatcher-worker-1
user integration 1 DefaultDispatcher-worker-2
finger scan 2 DefaultDispatcher-worker-1
user integration 2 DefaultDispatcher-worker-2
```
- هيتوقف بعد ثانيتين.

### إيه هو Global Scope؟
- الـ **Global Scope** شغال طول ما الأبليكيشن مفتوح.

### مثال الكود:
```kotlin
GlobalScope.launch(Dispatchers.IO) { waitFingerPrint() }
```

### note:
- ما تستخدموش كتير عشان ممكن يسبب **memory leaks** لو الكوروتين فضل شغال من غير داعي.


---
# 5. withContext
#### إيه هو withContext؟
هو tool في **Coroutines** بتخلّيك تبدّل الـ **Dispatcher** اللي الكوروتين بيشتغل عليه أثناء التنفيذ. يعني تقدر تقول لجزء معين من الكود: "اشتغل على **thread** معين" من غير ما تعقّد الكود.
- مفيد لما تكون بتعمل حاجة زي حفظ بيانات على ملف (بتحتاج **Dispatchers.IO**) وبعدين ترجع تعمل حاجة تانية على الـ **main thread** (زي تحديث الـ **UI**).
- الـ **Dispatcher** الواحد، زي **Dispatchers.IO** أو **Dispatchers.Default**، بيشتغل على أكتر من **worker thread** في الـ **Thread Pool**، يعني بيوزّع الشغل على **threads** كتير علشان يكون أسرع.

#### مثال بسيط:
تخيّل إنك بتعمل أبليكيشن، وعايز تحفظ داتا في ملف (ده شغل تقيل، يعني لازم **Dispatchers.IO**)، وبعدين ترجع تعمل حاجة تانية. من غير **withContext**، هتضطر تدير الـ **threads** بنفسك. لكن مع **withContext**، الكود بيبقى زي كده:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    coroutineScope.launch {
        println("Starting on ${Thread.currentThread().name}")
        withContext(Dispatchers.IO) {
            println("Saving data to file ${Thread.currentThread().name}")
        }
        println("Back to ${Thread.currentThread().name}")
    }
}
```

#### إيه اللي بيحصل هنا؟

- الـ **coroutineScope.launch** بيشغّل الكوروتين على الـ **Dispatcher** الافتراضي (غالبا الـ **main thread** أو الـ **Default** حسب الـ **scope**).
- لما نستخدم **withContext(Dispatchers.IO)**، الكود جواه (زي حفظ البيانات) بيتنفذ على **worker thread** من **Thread Pool** بتاع **Dispatchers.IO**.
- الـ **Dispatcher** زي **IO** بيستخدم أكتر من **worker thread**، فممكن تلاقي الـ task  بتشتغل على **IO-worker-1** أو **IO-worker-2**، حسب الـ **Thread Pool**.
- بعد ما **withContext** يخلّص، الكوروتين بيرجع للـ **Dispatcher** الأصلي (اللي بدأ بيه).

#### الـ Output 
```
Starting on main
Saving data to file DefaultDispatcher-worker-1
Back to main
```


---


#### 6. **launch**
- الـ **launch** هو **Coroutine Builder** بيشغّل كوروتين جديد، يعني زي ما تكون بتفتح تاسك لوحدها.
- بيرجّع **Job**، وده زي id للكوروتين، تقدر تتحكم فيها:
  - تقدر تعمل **cancel** للـ **Job** لو عايز توقّف الكوروتين.
  - تقدر تعمل **join** (هنشرحه تحت) علشان تستناها تخلّص.
  - تقدر تعمل **check** عليها، زي إنك تشوف الكوروتين شغال ولا لأ.
  - تقدر تحط الـ **Job** في **variable** علشان تتحكم فيه بعدين.

##### مثال بسيط:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        delay(1000)
        println("Task finished!")
    }
    println("Working now...")
    job.join() // استني التاسك يخلص
    println("Task done, let's continue!")
}
```

##### الـ Output:
```
Working now...
(بعد ثانية) Task finished!
Task done, let's continue!
```


#### 7. **join**
- الـ **join** بيخلّيك تستني الكوروتين اللي بتشتغل (اللي راجعة **Job**) تخلّص قبل ما تكمّل الكود اللي بعدها.
- يعني لو عندك كوروتين بتعمل حاجة مهمة، ومش عايز الكود اللي تحت يشتغل إلا لما تخلّص، تستخدم **join**.

##### مثال:
في المثال اللي فوق، لو ما استخدمناش `job.join()`، الكود هيكمّل على طول من غير ما يستني الكوروتين. لكن مع **join**، الكود بيستني.


#### 8. **async/await**
- الـ **async** زي **launch**، بس بيشغّل كوروتين ويرجّع **Deferred** بدل **Job**. الـ **Deferred** دي زي وعد إن الكوروتين هيرجّعلك نتيجة في المستقبل.
- علشان تجيب النتيجة، تستخدم **await**، وده بيستني الكوروتين تخلّص ويرجّعلك السطر الأخير من الكود.
- مفيد لما عايز ترجّع داتا من الكوروتين، زي إنك تحمّل داتا من النت.

##### مثال بسيط:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val deferred = async {
        delay(1000)
        "Data loaded!" // السطر ده هيرجع
    }
    println("Working now...")
    val result = deferred.await() // استني النتيجة
    println("Result: $result")
}
```

##### الـ Output:
```
Working now...
(بعد ثانية) Result: Data loaded!
```

#### 9. **Structured Concurrency**
- الـ **Structured Concurrency** هي طريقة **Coroutines** بتستخدمها علشان تنظّم الكوروتينز مع بعض.
- لو عندك **Job** (زي الـ parent ) بيشغّل **Job** تانية (زي الـ child)، لو عملت **cancel**  للـ parent ، الـ child هيتوقف تلقائيًا.
- يعني الكوروتينز بتبقى مربوطة ببعض، فلو الـ **scope** اتلغى، كل التاسك جواه بتتوقف، وده بيمنع **memory leaks**.

##### مثال بسيط:
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default)
    val parentJob = scope.launch {
        launch {
            delay(1000)
            println("Child task!")
        }
        println("Parent task!")
    }
    delay(500)
    parentJob.cancel() // الأب والابن هيتوقفوا
    println("Everything cancelled!")
}
```

##### الـ Output:
```
Parent task!
Everything cancelled!
```
- التاسك "Child" ما ظهرتش لأن الـ **parentJob** اتلغى قبل ما تخلّص.





#### 10. **Supervisor Coroutine**
- الـ **Supervisor Coroutine** هو نوع  من ال **Coroutine Scope** أو **Job** بيخلّي الكوروتينز ال child  تشتغل بشكل مستقل. يعني لو كوروتين child فشل (throw **exception**) أو اتلغى، ده ما بيأثرش على parent أو ال child  التانيين.


```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    supervisorScope {
        launch {
            delay(1000)
            throw Exception("Error in task 1!")
        }
        launch {
            delay(2000)
            println("Task 2 finished!")
        }
    }
    println("Scope completed!")
}
```

##### الـ Output:
```
(بعد ثانية) Exception: Error in task 1!
(بعد ثانيتين) Task 2 finished!
Scope completed!
```
- التاسك الأولى فشلت، بس التاسك التانية كمّلت شغلها عادي بسبب **supervisorScope**.



###  Error Handling with Coroutines 

#### المشكلة:
- لو حصل **Exception** في كوروتين، ممكن يوقّفها أو يخلّي الأبليكيشن يـ **crash** لو ما اتعالجش صح.
- الـ **Job** لو انتقلت لـ **Dispatcher** جديد (زي من **IO** لـ **Main**)، الـ **Exception** بينتقل معاها  ازم تهندله بـ **try-catch** جوا **withContext** أو بـ **CoroutineExceptionHandler**..
- **try-catch** عادي بيمنع الـ **crash**، بس لو الـ **Exception** في **parent Job**، كل الـ **child Jobs** هتتوقف بسبب **Structured Concurrency**.

#### إزاي نهندل الـ Exceptions؟
1. **try-catch**:
   - بيهندل **Exception** لكوروتين واحدة، بس ممكن يوقّف باقي الكوروتينز لو الـ **parent Job** فشلت.   ```

2. **CoroutineExceptionHandler**:
   - بيهندل **Exceptions** على مستوى الـ **Job** أو الـ **scope**، ويخلّي الكوروتينز التانية تكمّل.
   ```kotlin
   import kotlinx.coroutines.*
   
   fun main() = runBlocking {
       val handler = CoroutineExceptionHandler { _, e -> println("Caught: ${e.message}") }
       val scope = CoroutineScope(Dispatchers.Default + handler)
       scope.launch {
           throw Exception("Error in task!")
       }
       scope.launch {
           delay(1000)
           println("Other task finished!")
       }
       delay(2000)
   }
   ```
   **Output**:
   ```
   Caught: Error in task!
   Other task finished!
   ```

