<div dir="rtl" align="right">

# فصل ۷. تست‌نویسی

تست‌نویسی بخش ضروریِ کیفیت نرم‌افزار است. طرفداران تست واحد (Unit Testing) در سال‌های اخیر رایج شده‌اند؛ به نظر می‌رسد هرجا می‌خوانید یا می‌شنوید درباره‌اش صحبت می‌شود. بعضی‌ها توسعهٔ مبتنی بر تست (TDD) را ترویج می‌کنند، شیوه‌ای از کدنویسی که تضمین می‌کند وقتی برنامه کامل شد، مجموعهٔ جامعی از تست‌ها دارید. مزایای تست واحد برای کیفیت کد و زمان کلی انجام پروژه شناخته‌شده‌اند، اما با این حال بسیاری از توسعه‌دهندگان هنوز تست واحد نمی‌نویسند.

من تشویقتان می‌کنم حداقل بخشی از تست‌های واحد را بنویسید. از کدی شروع کنید که کمترین اعتماد را به آن دارید. به تجربهٔ من، تست‌های واحد دو مزیت اصلی به من داده‌اند:

- درک بهتر از کد. همان بخشی از برنامه که کار می‌کند ولی نمی‌دانید چطور؟ وقتی گزارش باگ‌های خیلی عجیب می‌رسد، همیشه گوشهٔ ذهنتان هست. نوشتن تست واحد برای کدی که برایتان سخت است، راه بسیار خوبی برای فهم روشن از نحوهٔ کار آن است. بعد از نوشتن تست‌های واحدی که رفتار آن را توصیف می‌کند، کد دیگر مرموز نیست؛ در نهایت مجموعه‌ای از تست‌های واحد دارید که رفتار آن و وابستگی‌هایش به سایر بخش‌های کد را توصیف می‌کند.
- اعتماد بیشتر برای اعمال تغییرات. دیر یا زود درخواست فیچری می‌آید که مجبور می‌شوید همان کد ترسناک را تغییر دهید و دیگر نمی‌توانید تظاهر کنید وجود ندارد (می‌دانم چه حسی دارد؛ تجربه کرده‌ام!). بهترین کار، پیش‌دستانه عمل کردن است: قبل از رسیدن درخواست فیچر، برای کد ترسناک تست واحد بنویسید. وقتی تست‌های واحدتان کامل شد، یک سیستم هشدار زودهنگام دارید که اگر تغییراتتان رفتار موجود را خراب کند بلافاصله به شما خبر می‌دهد. هنگام باز کردن Pull Request هم تست‌های واحد به شما اطمینان بیشتری می‌دهند که تغییرات کد، رفتار موجود را نمی‌شکنند.

هر دو مزیت دقیقاً به همان اندازه دربارهٔ کد خودتان صدق می‌کند که دربارهٔ کد دیگران. مطمئنم مزایای دیگری هم هست. آیا تست واحد فرکانس باگ‌ها را کاهش می‌دهد؟ به احتمال زیاد. آیا تست واحد زمان کلی پروژه را کم می‌کند؟ شاید. اما مزایایی که توصیف کردم قطعی‌اند؛ من هر بار که تست واحد می‌نویسم آن‌ها را تجربه می‌کنم. پس این هم معرفی و تشویق من برای تست واحد.

این فصل شامل دستورالعمل‌هایی است که همگی دربارهٔ تست‌نویسی هستند. خیلی از توسعه‌دهندگان (حتی آن‌هایی که معمولاً تست واحد می‌نویسند) از تست‌نویسی کد همزمان (Concurrent) دوری می‌کنند چون تصور می‌کنند سخت است. اما همان‌طور که این دستورالعمل‌ها نشان می‌دهند، تست‌نویسی کد همزمان آن‌قدرها هم که فکر می‌کنند دشوار نیست. قابلیت‌ها و کتابخانه‌های مدرن، مثل async و System.Reactive، توجه زیادی به موضوع تست داشته‌اند و این در عمل مشخص است. تشویقتان می‌کنم از این دستورالعمل‌ها برای نوشتن تست‌های واحد استفاده کنید، مخصوصاً اگر در همزمانی تازه‌کارید (یعنی کد همزمان جدید برایتان سخت یا ترسناک به نظر می‌رسد).

---

## ۷.۱ تست‌نویسی متدهای async

### مسئله
یک متد async دارید که باید برایش تست واحد بنویسید.

### راه‌حل
اغلب چارچوب‌های مدرن تست واحد از متدهای تستی `async Task` پشتیبانی می‌کنند؛ از جمله MSTest، NUnit و xUnit. MSTest از ویژوال استودیو ۲۰۱۲ پشتیبانی از این تست‌ها را آغاز کرد. اگر از چارچوب تست دیگری استفاده می‌کنید، ممکن است لازم باشد به آخرین نسخه ارتقا دهید. در اینجا یک نمونه از تست واحد async در MSTest آمده است:

<div dir="ltr" align="left">


```csharp
[TestMethod]
public async Task MyMethodAsync_ReturnsFalse()
{
    var objectUnderTest = ...;
    bool result = await objectUnderTest.MyMethodAsync();
    Assert.IsFalse(result);
}
```
</div>

چارچوب تست واحد متوجه می‌شود که نوع بازگشتی متد، `Task` است و هوشمندانه منتظر اتمام آن `Task` می‌ماند و بعد تست را «موفق» یا «ناموفق» علامت می‌زند.

اگر چارچوب تست واحد شما از تست‌های `async Task` پشتیبانی نمی‌کند، لازم است کمکش کنید تا برای عملیات ناهمگامِ تحت تست منتظر بماند. یک گزینه این است که از `GetAwaiter().GetResult()` برای بلاک کردن همگام روی `Task` استفاده کنید؛ اگر به جای `Wait()` از `GetAwaiter().GetResult()` استفاده کنید، در صورت وجود استثناء در `Task`، از پیچیده شدن آن در `AggregateException` جلوگیری می‌شود. با این حال، من ترجیح می‌دهم از نوع `AsyncContext` از پکیج NuGet به نام `Nito.AsyncEx` استفاده کنم:

<div dir="ltr" align="left">


```csharp
[TestMethod]
public void MyMethodAsync_ReturnsFalse()
{
    AsyncContext.Run(async () =>
    {
        var objectUnderTest = ...;
        bool result = await objectUnderTest.MyMethodAsync();
        Assert.IsFalse(result);
    });
}
```

</div>

`AsyncContext.Run` صبر می‌کند تا همهٔ متدهای ناهمگام کامل شوند.

### بحث
شبیه‌سازی (Mock) وابستگی‌های ناهمگام در ابتدا کمی دست‌وپاگیر است. خوب است حداقل واکنش متدهایتان را به این حالات تست کنید: موفقیت همگام (Mock با `Task.FromResult`)، خطای همگام (Mock با `Task.FromException`) و موفقیت ناهمگام (Mock با `Task.Yield` و یک مقدار بازگشتی). پوشش `Task.FromResult` و `Task.FromException` را در دستورالعمل ۲.۲ خواهید یافت. از `Task.Yield` می‌توان برای اجبار به رفتار ناهمگام استفاده کرد و عمدتاً در تست‌های واحد مفید است:

<div dir="ltr" align="left">


```csharp
interface IMyInterface
{
    Task<int> SomethingAsync();
}

class SynchronousSuccess : IMyInterface
{
    public Task<int> SomethingAsync()
    {
        return Task.FromResult(13);
    }
}

class SynchronousError : IMyInterface
{
    public Task<int> SomethingAsync()
    {
        return Task.FromException<int>(new InvalidOperationException());
    }
}

class AsynchronousSuccess : IMyInterface
{
    public async Task<int> SomethingAsync()
    {
        await Task.Yield(); // اجبار به رفتار ناهمگام.
        return 13;
    }
}
```

</div>

هنگام تست کد ناهمگام، بن‌بست‌ها (deadlock) و شرایط رقابتی (race conditions) ممکن است بیشتر از تست کد همگام رخ نمایان کنند. به نظر من تنظیم «Timeout» به‌ازای هر تست مفید است؛ در ویژوال استودیو می‌توانید یک فایل تنظیمات تست به راه‌حل‌تان اضافه کنید که امکان تعیین Timeout برای هر تست را می‌دهد. مقدار پیش‌فرض نسبتاً زیاد است؛ من معمولاً Timeout هر تست را روی دو ثانیه می‌گذارم.

> نکته: نوع `AsyncContext` در پکیج NuGet به نام `Nito.AsyncEx` قرار دارد.

> مطالعهٔ بیشتر: دستورالعمل ۷.۲ به تست‌نویسی متدهای ناهمگامی می‌پردازد که انتظار می‌رود شکست بخورند.

---

## ۷.۲ تست‌نویسی متدهای async که انتظار می‌رود شکست بخورند

### مسئله
باید تست واحدی بنویسید که یک شکست مشخص را برای متد `async Task` بررسی کند.

### راه‌حل
اگر توسعهٔ دسکتاپ یا سمت سرور انجام می‌دهید، MSTest از تست شکست با استفاده از `ExpectedExceptionAttribute` معمول پشتیبانی می‌کند:

<div dir="ltr" align="left">


```csharp
// راه‌حل پیشنهادی نیست؛ توضیح در ادامه.
[TestMethod]
[ExpectedException(typeof(DivideByZeroException))]
public async Task Divide_WhenDenominatorIsZero_ThrowsDivideByZero()
{
    await MyClass.DivideAsync(4, 0);
}
```

</div>

با این حال، این راه‌حل بهترین نیست: `ExpectedException` در واقع طراحی ضعیفی دارد. استثنایی که انتظارش می‌رود ممکن است توسط هر یک از متدهایی که در متد تست شما فراخوانی می‌شوند پرتاب شود. طراحی بهتر این است که بررسی کنید یک بخش مشخص از کد آن استثناء را پرتاب می‌کند، نه کل تست واحد.

اغلب چارچوب‌های مدرن تست واحد، `Assert.ThrowsAsync<TException>` را به شکلی ارائه می‌کنند. برای مثال، می‌توانید از `ThrowsAsync` در xUnit این‌طور استفاده کنید:

<div dir="ltr" align="left">


```csharp
[Fact]
public async Task Divide_WhenDenominatorIsZero_ThrowsDivideByZero()
{
    await Assert.ThrowsAsync<DivideByZeroException>(async () =>
    {
        await MyClass.DivideAsync(4, 0);
    });
}
```

</div>

> هشدار: فراموش نکنید که Task بازگشتی از `ThrowsAsync` را `await` کنید! این `await` هرگونه شکست Assertion را که شناسایی شود منتشر می‌کند. اگر `await` را فراموش کنید و هشدار کامپایلر را نادیده بگیرید، تست واحد شما همیشه بی‌سروصدا موفق می‌شود، فارغ از رفتار متدتان.

متأسفانه چند چارچوب دیگر تست واحد معادل سازگار با async برای `ThrowsAsync` ندارند. اگر در چنین وضعیتی هستید، خودتان یکی بسازید:

<div dir="ltr" align="left">


```csharp
/// <summary>
/// اطمینان حاصل می‌کند که یک Delegate ناهمگام استثناء پرتاب می‌کند.
/// </summary>
/// <typeparam name="TException">نوع استثنایی که انتظار می‌رود.</typeparam>
/// <param name="action">Delegate ناهمگامی که باید تست شود.</param>
/// <param name="allowDerivedTypes">آیا انواع مشتق‌شده پذیرفته شوند؟</param>
public static async Task<TException> ThrowsAsync<TException>(
    Func<Task> action,
    bool allowDerivedTypes = true)
    where TException : Exception
{
    try
    {
        await action();
        var name = typeof(TException).Name;
        Assert.Fail($"Delegate did not throw expected exception {name}.");
        return null;
    }
    catch (Exception ex)
    {
        if (allowDerivedTypes && !(ex is TException))
            Assert.Fail($"Delegate threw exception of type {ex.GetType().Name}, " +
                        $"but {typeof(TException).Name} or a derived type was expected.");
        if (!allowDerivedTypes && ex.GetType() != typeof(TException))
            Assert.Fail($"Delegate threw exception of type {ex.GetType().Name}, " +
                        $"but {typeof(TException).Name} was expected.");
        return (TException)ex;
    }
}
```

</div>

می‌توانید از این متد دقیقاً مانند سایر متدهای `Assert.ThrowsAsync<TException>` استفاده کنید. فراموش نکنید مقدار بازگشتی را `await` کنید!

### بحث
تست‌کردن مدیریت خطا به اندازهٔ تست سناریوهای موفق اهمیت دارد. بعضی حتی می‌گویند مهم‌تر است، چون سناریوی موفق همان است که همه قبل از انتشار نرم‌افزار امتحانش می‌کنند. اگر برنامهٔ شما رفتار عجیبی داشته باشد، به دلیل وضعیت خطای غیرمنتظره خواهد بود.

با این حال، من توسعه‌دهندگان را تشویق می‌کنم از `ExpectedException` فاصله بگیرند. بهتر است استثنایی را که در یک نقطهٔ مشخص پرتاب می‌شود تست کنید تا این‌که انتظار استثناء را در هر زمانی از طول تست داشته باشید. به‌جای `ExpectedException` از `ThrowsAsync` (یا معادلش در چارچوب تست شما) استفاده کنید، یا از پیاده‌سازی `ThrowsAsync` مشابه نمونهٔ بالا استفاده نمایید.

> مطالعهٔ بیشتر: دستورالعمل ۷.۱ اصول تست‌نویسی متدهای ناهمگام را پوشش می‌دهد.

</div>




<div dir="rtl" style="direction: rtl; text-align: right;">

# بخش 7 — تست‌نویسی اجزاء ناهمگام و واکنشی

## 7.3 تست‌نویسی متدهای `async void`

### مسئله
یک متد `async void` دارید که باید برای آن تست واحد بنویسید.

### راه‌حل (خلاصه)
توقف. به‌جای حل این مسئله، باید نهایت تلاش خود را بکنید تا از `async void` جلوگیری کنید. اگر ممکن است متد `async void` را به `async Task` تغییر دهید، این کار را انجام دهید.

اگر متد شما مجبور است `async void` باشد (مثلاً برای تطابق با امضای یک اینترفیس)، بهتر است این الگو را دنبال کنید:
- یک متد `async Task` بسازید که تمام منطق را در بر داشته باشد.
- یک پوشانندهٔ `async void` بسازید که فقط متد `async Task` را صدا می‌زند و منتظر نتیجهٔ آن می‌ماند.

به این ترتیب، متد `async void` نیازهای معماری را پوشش می‌دهد و متد `async Task` قابل تست خواهد بود.

### تست‌نویسی در صورتی که تغییر ممکن نباشد
اگر تغییر متد غیرممکن است و مجبورید برای یک متد `async void` تست بنویسید، می‌توانید از کلاس `AsyncContext` در کتابخانهٔ Nito.AsyncEx استفاده کنید. توجه کنید که این راه‌حل پیشنهادی اصلی نیست؛ بازآرایی کد همیشه ارجح است.


<div dir="ltr" align="left">

نمونه:
```csharp


// راه‌حل پیشنهادی نیست؛ ادامهٔ این بخش را ببینید.
[TestMethod]
public void MyMethodAsync_DoesNotThrow()
{
    AsyncContext.Run(() =>
    {
        var objectUnderTest = new Sut(); // ...;
        objectUnderTest.MyVoidMethodAsync();
    });
}
```
</div>

نوع `AsyncContext` صبر می‌کند تا همهٔ عملیات‌های ناهمگام کامل شوند (از جمله متدهای `async void`) و استثناهایی را که آن‌ها پرتاب می‌کنند منتشر می‌کند.

نکته: نوع `AsyncContext` در پکیج NuGet به نام `Nito.AsyncEx` قرار دارد.

### بحث
یکی از رهنمودهای کلیدی در کدنویسی async این است که از `async void` اجتناب کنید. اکیداً توصیه می‌کنم به‌جای استفاده از `AsyncContext` برای تست‌نویسی متدهای `async void`، کد خود را بازآرایی (refactor) کنید.

مطالعهٔ بیشتر: دستورالعمل 7.1 تست‌نویسی متدهای `async Task` را پوشش می‌دهد.

---

## 7.4 تست‌نویسی مش‌های Dataflow

### مسئله
در برنامهٔ خود یک بلاک (مش) Dataflow دارید و باید صحت عملکرد آن را راستی‌آزمایی کنید.

### راه‌حل
مش‌های Dataflow مستقل و ذاتاً ناهمگام‌اند. بنابراین طبیعی‌ترین روش برای تست، نوشتن تست‌های واحد ناهمگام است.

نمونهٔ تست برای یک بلاک سفارشی:

<div dir="ltr" align="left">


```csharp
[TestMethod]
public async Task MyCustomBlock_AddsOneToDataItems()
{
    var myCustomBlock = CreateMyCustomBlock();
    myCustomBlock.Post(3);
    myCustomBlock.Post(13);
    myCustomBlock.Complete();
    Assert.AreEqual(4, myCustomBlock.Receive());
    Assert.AreEqual(14, myCustomBlock.Receive());
    await myCustomBlock.Completion;
}
```

</div>

### تست‌نویسی خطاها
وقتی خطاها مطرح باشند، تست‌ها کمی پیچیده‌تر می‌شوند؛ استثناها در مش‌های Dataflow ممکن است با `AggregateException` پوشش داده شوند. مثال زیر نشان می‌دهد چگونه با خطا برخورد کنیم:

<div dir="ltr" align="left">

```csharp



[TestMethod]
public async Task MyCustomBlock_Fault_DiscardsDataAndFaults()
{
    var myCustomBlock = CreateMyCustomBlock();
    myCustomBlock.Post(3);
    myCustomBlock.Post(13);
    (myCustomBlock as IDataflowBlock).Fault(new InvalidOperationException());
    try
    {
        await myCustomBlock.Completion;
    }
    catch (AggregateException ex)
    {
        AssertExceptionIs<InvalidOperationException>(
            ex.Flatten().InnerException, false);
    }
}

public static void AssertExceptionIs<TException>(Exception ex,
    bool allowDerivedTypes = true)
{
    if (allowDerivedTypes && !(ex is TException))
        Assert.Fail("Exception is of type " + ex.GetType().Name + ", but "
            + typeof(TException).Name + " or a derived type was expected.");
    if (!allowDerivedTypes && ex.GetType() != typeof(TException))
        Assert.Fail("Exception is of type " + ex.GetType().Name + ", but "
            + typeof(TException).Name + " was expected.");
}
```
</div>

### بحث
تست‌نویسی مستقیم بلاک‌های Dataflow شدنی اما دست‌وپاگیر است. اگر بلاک شما بخشی از یک مؤلفهٔ بزرگ‌تر است، ممکن است ساده‌تر باشد که مؤلفهٔ بزرگ‌تر را تست کنید و بدین ترتیب بلاک نیز به‌طور ضمنی تست شده باشد. اما اگر دارید یک بلاک قابل‌استفادهٔ مجدد توسعه می‌دهید، تست‌های واحد مستقل مشابه نمونه‌های بالا لازم‌اند.

مطالعهٔ بیشتر: دستورالعمل 7.1 (تست متدهای async).

---

## 7.5 تست‌نویسی System.Reactive (Observables)

### مسئله
بخشی از برنامهٔ شما از `IObservable<T>` استفاده می‌کند و باید آن را تست کنید.

### راه‌حل
System.Reactive عملگرهای ساخت و تبدیل متنوعی دارد:
- برای ساخت Stub از وابستگی‌های observable می‌توان از عملگرهایی مانند `Return` استفاده کرد.
- برای تبدیل observable به یک مقدار واحد یا مجموعه می‌توان از عملگرهایی مانند `SingleAsync` استفاده کرد.

نمونه: کلاس سرویس HTTP و کاربرد Timeout

<div dir="ltr" align="left">

```csharp



public interface IHttpService
{
    IObservable<string> GetString(string url);
}

public class MyTimeoutClass
{
    private readonly IHttpService _httpService;
    public MyTimeoutClass(IHttpService httpService)
    {
        _httpService = httpService;
    }
    public IObservable<string> GetStringWithTimeout(string url)
    {
        return _httpService.GetString(url)
            .Timeout(TimeSpan.FromSeconds(1));
    }
}

```



نمونهٔ Stub موفق:
```csharp
class SuccessHttpServiceStub : IHttpService
{
    public IObservable<string> GetString(string url)
    {
        return Observable.Return("stub");
    }
}

[TestMethod]
public async Task MyTimeoutClass_SuccessfulGet_ReturnsResult()
{
    var stub = new SuccessHttpServiceStub();
    var my = new MyTimeoutClass(stub);
    var result = await my.GetStringWithTimeout("http://www.example.com/")
        .SingleAsync();
    Assert.AreEqual("stub", result);
}
```

نمونهٔ Stub خطا (با استفاده از `Throw`) و تستِ انتشار خطا:
```csharp
private class FailureHttpServiceStub : IHttpService
{
    public IObservable<string> GetString(string url)
    {
        return Observable.Throw<string>(new HttpRequestException());
    }
}

[TestMethod]
public async Task MyTimeoutClass_FailedGet_PropagatesFailure()
{
    var stub = new FailureHttpServiceStub();
    var my = new MyTimeoutClass(stub);
    await ThrowsAsync<HttpRequestException>(async () =>
    {
        await my.GetStringWithTimeout("http://www.example.com/")
            .SingleAsync();
    });
}
```

</div>

<div dir="rtl" align="right">


### بحث
 `Return` و `Throw` برای ساخت Stubهای سادهٔ observable عالی‌اند.

 `SingleAsync` راه ساده‌ای برای await کردن نتایج observable در تست‌های async فراهم می‌کند.

 مشکل زمانی که وابستگی‌ها به زمان واقعی (مثل Timeout) بستگی دارند: تست ممکن است زمان واقعی را منتظر بماند و باعث شود تست‌ها ناپایدار و کند شوند. این روش باعث بروز race 
condition و مشکلات مقیاس‌پذیری در مجموعهٔ تست‌ها می‌شود.


 برای تست دنباله‌های وابسته به زمان بهتر است از روش‌هایی استفاده شود که امکان Stub کردن زمان را فراهم می‌آورند — دستورالعمل 7.6 به این موضوع می‌پردازد.

مطالعهٔ بیشتر:
- دستورالعمل 7.1: تست‌نویسی متدهای `async`
- دستورالعمل 7.6: تست‌نویسی دنباله‌های observable وابسته به گذر زمان

</div>

</div>



<div dir="rtl" style="direction: rtl; text-align: right;">

# بخش 7.6 — تست‌نویسی System.Reactive Observables با زمان‌بندی ساختگی (Faked Scheduling)

## مسئله
یک observable دارید که به زمان وابسته است و می‌خواهید تست واحدی بنویسید که به زمان واقعی وابسته نباشد. Observableهایی که به زمان وابسته‌اند شامل مواردی می‌شوند که از timeout، windowing/buffering و throttling/sampling استفاده می‌کنند. هدف این است که آنها را تست کنید بدون اینکه زمان اجرای تست‌ها بیش از حد طولانی شود.

## راه‌حل (خلاصه)
قرار دادن تأخیر واقعی در تست‌ها ممکن است کار کند اما دو مشکل دارد:
1. تست‌ها زمان‌بر می‌شوند.
2. به دلیل اجرای همزمان تست‌ها، شرایط رقابتی (race conditions) و زمان‌بندی غیرقابل‌پیش‌بینی رخ می‌دهد.

کتابخانهٔ System.Reactive (Rx) برای قابل‌تست‌بودن طراحی شده است: هر عملگر وابسته به زمان با استفاده از یک انتزاع به نام زمان‌بند (IScheduler) پیاده‌سازی شده است. برای آزمون‌پذیر کردن observables، فراخواننده باید بتواند زمان‌بند را تعیین کند.

مثال: افزودن پارامتر زمان‌بند به کلاس MyTimeoutClass (دستورالعمل 7.5)

<div dir="ltr" align="left">


```csharp
public interface IHttpService
{
    IObservable<string> GetString(string url);
}

public class MyTimeoutClass
{
    private readonly IHttpService _httpService;
    public MyTimeoutClass(IHttpService httpService)
    {
        _httpService = httpService;
    }
    public IObservable<string> GetStringWithTimeout(string url,
        IScheduler scheduler = null)
    {
        return _httpService.GetString(url)
            .Timeout(TimeSpan.FromSeconds(1), scheduler ?? Scheduler.Default);
    }
}
```

سپس Stub سرویس HTTP را طوری بسازید که زمان‌بند را بپذیرد و یک تأخیر متغیر اعمال کند:
```csharp
private class SuccessHttpServiceStub : IHttpService
{
    public IScheduler Scheduler { get; set; }
    public TimeSpan Delay { get; set; }
    public IObservable<string> GetString(string url)
    {
        return Observable.Return("stub")
            .Delay(Delay, Scheduler);
    }
}
```
</div>

حالا از `TestScheduler` استفاده کنید — کلاسی از بستهٔ Microsoft.Reactive.Testing که زمان مجازی را کنترل می‌کند.

> نکته: `TestScheduler` در پکیج NuGet جداگانه‌ای قرار دارد: `Microsoft.Reactive.Testing`.


### نمونهٔ موفق (تاخیر کوتاه)

<div dir="ltr" align="left">


```csharp
[TestMethod]
public void MyTimeoutClass_SuccessfulGetShortDelay_ReturnsResult()
{
    var scheduler = new TestScheduler();
    var stub = new SuccessHttpServiceStub
    {
        Scheduler = scheduler,
        Delay = TimeSpan.FromSeconds(0.5),
    };
    var my = new MyTimeoutClass(stub);
    string result = null;
    my.GetStringWithTimeout("http://www.example.com/", scheduler)
        .Subscribe(r => { result = r; });
    scheduler.Start();
    Assert.AreEqual("stub", result);
}
```
</div>

این تست یک تاخیر نیم‌ثانیه‌ای را در زمان مجازی شبیه‌سازی می‌کند؛ اجرای واقعی تست بسیار سریع‌تر (مثلاً ~70ms) انجام می‌شود.



### تست timeout (تاخیر طولانی‌تر)

<div dir="ltr" align="left">


```csharp
[TestMethod]
public void MyTimeoutClass_SuccessfulGetLongDelay_ThrowsTimeoutException()
{
    var scheduler = new TestScheduler();
    var stub = new SuccessHttpServiceStub
    {
        Scheduler = scheduler,
        Delay = TimeSpan.FromSeconds(1.5),
    };
    var my = new MyTimeoutClass(stub);
    Exception result = null;
    my.GetStringWithTimeout("http://www.example.com/", scheduler)
        .Subscribe(_ => Assert.Fail("Received value"),
                   ex => { result = ex; });
    scheduler.Start();
    Assert.IsInstanceOfType(result, typeof(TimeoutException));
}

```

</div>

با زمان مجازی، این تست هم فوراً اجرا می‌شود و زمان واقعی صرف نشده است.

## بحث

- توصیهٔ عملی: سعی کنید هر تست فقط یک رفتار را بررسی کند. برای تست timeout می‌توانید:
  - یک تست که بررسی کند timeout زودتر رخ نمی‌دهد (قبل از زمان مورد نظر).
  - یک تست که بررسی کند بعد از عبور زمان، timeout رخ می‌دهد.
- هم‌زمان با پیاده‌سازی کدهای System.Reactive، از ابتدا تست‌نویسی را آغاز کنید تا با افزایش پیچیدگی کد، از قابلیت‌های Microsoft.Reactive.Testing بهره‌مند شوید.

## مطالعهٔ بیشتر
- دستورالعمل 7.5: مبانی تست‌نویسی دنباله‌های observable
</div>
