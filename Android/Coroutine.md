‏لما بنعمل أبليكيشن، اوقات بنحتاج نعمل حاجة بتاخد وقت، زي إنك تحمّل صورة من النت. وفي نفس الوقت، عايزين حاجة تانية زي إن الـ **user** يضغط  علي button ويحصل رد فعل على طول. لو استخدمنا الـ **main thread** أو الـ **worker threads** في الـ **Thread Pool**، هيحصل مشاكل:
 لان لازم ننظّم الشغل بين الـ **threads** بنفسنا (ده اسمه **synchronization**)، وده بيخلّي الكود  مش مفهوم.
 ولو استخدمنا **callbacks**، الكود بيبقى معقد أكتر.
ممكن يحصل مشاكل زي:
الـ Deadlocks: لما البرنامج يعلّق وما يكمّلش.
الـ Race conditions: لما الحاجات تشتغل بطريقة غلط.    
لو الـ **thread switching** كتير، البرنامج بيستهلك موارد زيادة (**resource consuming**).

هنا بقى دخلت **Coroutines**، ودي **tool** في **Kotlin** بتخلّيك تشغّل الحاجات دي مع بعض بطريقة سهلة ومنظمة، من غير ما تحتاج تدير الـ **threads** بنفسك. يعني بدل ما تعقّد الكود، **Coroutines** بتخلّيه بسيط ومرتب



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
- زي فيلم **رضا** لما كانوا في سباق جري، واحد (**suspend fun**) بيجري، لو تعب يقف ويخرج، يدخل التاني، ولو التاني وقف يدخل التالت، وهكذا بيفضلوا يبدّلوا لحد النهاية.
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
1. **Dispatchers.IO**: للشغل التقيل زي الـ **network** أو الـ **database**. بيستخدم **Thread Pool** مخصص.
2. **Dispatchers.Main**: للـ **Main Thread**، زي تحديث الـ **UI** في أندرويد.
3. **Dispatchers.Default**: للحوسبة التقيلة زي معالجة بيانات. بيستخدم **Thread Pool** حسب قوة الـ **CPU**.
4. **Dispatchers.Unconfined**: بيشتغل على أي **thread** متاح، بس قليل نستخدمه.

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
تمام، هكتب ملخص عن **withContext** بالعامية المصرية، بأسلوب إنستراكتور بيشرح في كورس تطوير تطبيقات للديفيلوبرز على **Kotlin** وأندرويد، بس بطريقة بسيطة و**readable** تناسب حتى اللي ما يعرفش حاجة عن الموضوع. هتكون مركزة على إزاي **withContext** بيخلّيك تبدّل بين الـ **Dispatchers**، وإن الـ **Dispatcher** الواحد بيشتغل على أكتر من **worker thread**. هحافظ على المصطلحات التقنية زي **Dispatcher** و**worker threads**، وهظبّط الكود اللي بعتيه مع تنسيق واضح.

---
# withContext
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
- الـ **Dispatcher** زي **IO** بيستخدم أكتر من **worker thread**، فممكن تلاقي المهمة بتشتغل على **IO-worker-1** أو **IO-worker-2**، حسب الـ **Thread Pool**.
- بعد ما **withContext** يخلّص، الكوروتين بيرجع للـ **Dispatcher** الأصلي (اللي بدأ بيه).

#### الـ Output 
```
Starting on main
Saving data to file DefaultDispatcher-worker-1
Back to main
```


