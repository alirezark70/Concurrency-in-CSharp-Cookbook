

<div dir="rtl" align="Right">

# فصل ۲. اصول Async

این فصل، شما را با اصول اولیه استفاده از async و await برای عملیات‌های ناهمگام آشنا می‌کند.  
در این فصل، فقط با عملیات‌هایی سر و کار داریم که ذاتاً ناهمگام هستند؛ مانند درخواست‌های HTTP، دستورات پایگاه داده و فراخوانی سرویس‌های وب.

اگر عملیاتی دارید که مصرف CPU بالایی دارد اما می‌خواهید مثل یک عملیات ناهمگام با آن رفتار کنید (مثلاً برای آنکه thread رابط کاربری را مسدود نکند)، به فصل ۴ و دستور ۸.۴ مراجعه کنید. همچنین این فصل فقط به عملیاتی می‌پردازد که یک بار شروع و یک بار تمام می‌شوند؛ اگر باید جریان‌هایی از رویدادها را مدیریت کنید، به فصل‌های ۳ و ۶ مراجعه کنید.

---

## ۲.۱ مکث (Pause) برای یک بازه زمانی

### مسئله

باید (به صورت ناهمگام) برای مدت زمانی منتظر بمانید.  
این وضعیت در سناریوهای تست واحد یا پیاده‌سازی تاخیرهای تکرار (retry delay) رایج است. همچنین هنگام کدنویسی timeoutهای ساده به کار می‌آید.

### راه‌حل

نوع (Type) `Task` یک متد ایستا به نام `Delay` دارد که یک task باز می‌گرداند که پس از مدت زمان مشخص شده کامل می‌شود.

کد نمونه زیر یک task تعریف می‌کند که به صورت ناهمگام تکمیل می‌شود.  
هنگام شبیه‌سازی یک عملیات ناهمگام، مهم است که هم موفقیت همزمان (synchronous success)، هم موفقیت ناهمگام و هم شکست ناهمگام را تست کنید. نمونه زیر یک task برای حالت موفقیت ناهمگام بازمی‌گرداند:

<div dir="ltr" align="left">

```csharp
async Task<T> DelayResult<T>(T result, TimeSpan delay)
{
    await Task.Delay(delay);
    return result;
}
```

</div>

---

### Backoff نمایی (exponential backoff)

راهبردی است که در آن با هر بار تلاش مجدد، بازه‌های تاخیر افزایش می‌یابد.  
این روش را هنگام کار با سرویس‌های وب به کار برید تا سرور با تعداد زیاد retry بمباران نشود.  
مثال بعدی یک پیاده‌سازی ساده از backoff نمایی است:

<div dir="ltr" align="left">

```csharp
async Task<string> DownloadStringWithRetries(HttpClient client, string uri)
{
    // بار اول بعد از ۱ ثانیه، سپس ۲ ثانیه و بعد ۴ ثانیه تلاش مجدد کند.
    TimeSpan nextDelay = TimeSpan.FromSeconds(1);
    for (int i = 0; i != 3; ++i)
    {
        try
        {
            return await client.GetStringAsync(uri);
        }
        catch
        {
        }
        await Task.Delay(nextDelay);
        nextDelay = nextDelay + nextDelay;
    }
    // یک بار آخر تلاش کند و اجازه دهد خطا propagate شود.
    return await client.GetStringAsync(uri);
}
```

</div>

---

### نکته

برای کدهای تولیدی (production)، یک راهکار کامل‌تر مثل کتابخانه Polly (از طریق NuGet) توصیه می‌شود؛  
این کد صرفاً یک مثال ساده از کاربرد `Task.Delay` است.

</div>




## استفاده از Task.Delay برای پیاده‌سازی تایم‌اوت

شما همچنین می‌توانید از `Task.Delay` به عنوان یک تایم‌اوت ساده استفاده کنید.

معمولاً برای پیاده‌سازی تایم‌اوت از نوع `CancellationTokenSource` استفاده می‌شود (دستور ۱۰.۳).  
می‌توانید یک توکن لغو را درون یک `Task.Delay` بی‌نهایت قرار دهید تا یک task داشته باشید که بعد از مدت مشخص لغو شود.  
در نهایت این تایمر را با `Task.WhenAny` (دستور ۲.۵) ترکیب کنید تا یک «تایم‌اوت نرم» بسازید.

مثال زیر اگر سرویس ظرف سه ثانیه پاسخ ندهد، مقدار `null` باز می‌گرداند:

<div dir="ltr" align="left">

```csharp
async Task<string> DownloadStringWithTimeout(HttpClient client, string uri)
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(3));
    Task<string> downloadTask = client.GetStringAsync(uri);
    Task timeoutTask = Task.Delay(Timeout.InfiniteTimeSpan, cts.Token);
    Task completedTask = await Task.WhenAny(downloadTask, timeoutTask);
    if (completedTask == timeoutTask)
        return null;
    return await downloadTask;
}
```

</div>

در حالی که می‌توان از `Task.Delay` به عنوان یک تایم‌اوت «نرم» استفاده کرد، اما این روش محدودیت‌هایی دارد.  
اگر عملیات به تایم‌اوت برسد، عملیات لغو نمی‌شود؛ در مثال بالا، وظیفه دانلود همچنان ادامه می‌یابد و پاسخ کامل را دانلود می‌کند، حتی اگر نتیجه دیگر مورد نیاز نباشد!  
راه‌حل بهتر این است که از `CancellationToken` به عنوان تایم‌اوت استفاده کنید و آن را مستقیماً به عملیات (مثلاً به `GetStringAsync`) ارسال نمایید.  
البته، گاهی اوقات عملیات قابل لغو نیست و در این حالت، شاید چاره‌ای جز استفاده از `Task.Delay` برای شبیه‌سازی تایم‌اوت نداشته باشید.

---

## بحث

استفاده از `Task.Delay` انتخاب خوبی برای تست واحد کدهای ناهمگام یا پیاده‌سازی منطق تلاش مجدد (retry) است.  
اما اگر واقعا نیاز به تایم‌اوت دارید، معمولاً `CancellationToken` گزینه مناسب‌تری است.

---

## همچنین ببینید

- **دستور ۲.۵:** نحوه استفاده از `Task.WhenAny` برای تشخیص اینکه کدام task زودتر تکمیل می‌شود را توضیح می‌دهد.
- **دستور ۱۰.۳:** به‌کارگیری `CancellationToken` به عنوان تایم‌اوت را پوشش می‌دهد.

## ۲.۲ بازگرداندن وظایف تکمیل‌شده (Completed Tasks)

### مسئله

گاهی لازم است یک متد همزمان (synchronous) را با امضای (signature) ناهمگام (asynchronous) پیاده‌سازی کنید.  
این وضعیت زمانی پیش می‌آید که از یک رابط یا کلاس پایه‌ی ناهمگام ارث‌بری می‌کنید اما می‌خواهید آن را همزمان پیاده‌سازی کنید.  
این تکنیک به‌ویژه هنگام تست واحد کد ناهمگام (برای ساخت استاب یا ماک ساده روی یک اینترفیس ناهمگام) مفید است.

---

### راه‌حل

می‌توانید از متد `Task.FromResult` استفاده کنید تا یک `Task<T>` که با مقدار مورد نظر بلافاصله تکمیل شده بسازید و بازگردانید:

<div dir="ltr" align="left">

```csharp
interface IMyAsyncInterface
{
    Task<int> GetValueAsync();
}

class MySynchronousImplementation : IMyAsyncInterface
{
    public Task<int> GetValueAsync()
    {
        return Task.FromResult(13);
    }
}
```

</div>

برای متدهایی که خروجی ندارند، می‌توانید از `Task.CompletedTask` استفاده کنید.  
این یک Task کش‌شده است که با موفقیت کامل شده:

<div dir="ltr" align="left">

```csharp
interface IMyAsyncInterface
{
    Task DoSomethingAsync();
}

class MySynchronousImplementation : IMyAsyncInterface
{
    public Task DoSomethingAsync()
    {
        return Task.CompletedTask;
    }
}
```

</div>

متد `Task.FromResult` تنها برای حالت بازگشت موفق کاربرد دارد.  
اگر نیاز به برگرداندن Taskی دارید که با نوع دیگری از نتیجه تمام می‌شود (مثلاً Taskی که با استثنا کامل می‌شود)، می‌توانید از `Task.FromException` استفاده کنید:

<div dir="ltr" align="left">

```csharp
Task<T> NotImplementedAsync<T>()
{
    return Task.FromException<T>(new NotImplementedException());
}
```

</div>

همچنین متدی به نام `Task.FromCanceled` وجود دارد تا Taskی بسازید که از همان ابتدا با `CancellationToken` لغو شده باشد:

<div dir="ltr">

```csharp
Task<int> GetValueAsync(CancellationToken cancellationToken)
{
    if (cancellationToken.IsCancellationRequested)
        return Task.FromCanceled<int>(cancellationToken);

    return Task.FromResult(13);
}
```

</div>

اگر احتمال شکست اجرای همزمان وجود دارد، باید استثناها را گرفته و با `Task.FromException` برگردانید:

<div dir="ltr" align="left">

```csharp
interface IMyAsyncInterface
{
    Task DoSomethingAsync();
}

class MySynchronousImplementation : IMyAsyncInterface
{
    public Task DoSomethingAsync()
    {
        try
        {
            DoSomethingSynchronously();
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            return Task.FromException(ex);
        }
    }
}
```

</div>

---

### بحث

اگر یک اینترفیس ناهمگام را با کد همزمان پیاده‌سازی می‌کنید، از هرگونه بلاک‌کردن (Blocking) اجتناب کنید.  
ایده‌آل نیست که یک متد ناهمگام را که می‌تواند واقعاً ناهمگام باشد، بلاک کنید و سپس Task تکمیل شده بازگردانید.  
یک مثال منفی، خواننده‌های متنی Console در .NET BCL است:

`Console.In.ReadLineAsync`  
در واقع thread فراخوان را تا وقتی خط خوانده شود مسدود می‌کند، و پس از آن task تکمیل شده بازمی‌گرداند.  
این رفتار شهودی نیست و باعث سردرگمی بسیاری از توسعه‌دهندگان شده است.  
اگر یک متد ناهمگام بلاک کند، سایر وظایف (task) نمی‌توانند اجرا شوند، که مانع همروندی و حتی در برخی موارد باعث بن‌بست می‌شود.

اگر مرتباً از `Task.FromResult` با یک مقدار خاص استفاده می‌کنید، می‌توانید همان Task را کش کنید.  
مثلاً اگر چندبار نیاز است `Task<int>` با مقدار صفر بسازید، کافیست یکی بسازید و باقی موارد همان را بازگردانید:

<div dir="ltr" align="left">

```csharp
private static readonly Task<int> zeroTask = Task.FromResult(0);

Task<int> GetValueAsync()
{
    return zeroTask;
}
```

</div>

در واقع،  
`Task.FromResult`،  
`Task.FromException`،  
و  
`Task.FromCanceled`  
همه تنها میان‌بر برای ساخت سریع taskهای تکمیل‌شده‌اند.

ابزار سطح پایین‌تر و منعطف‌تر برای این کار  
`TaskCompletionSource<T>`  
است که برای تعامل عمومی با سایر شیوه‌های کدنویسی ناهمگام به‌کار می‌رود.  
معمولاً اگر می‌خواهید task تکمیل‌شده بازگردانید سراغ این میان‌برها بروید.  
اگر لازم است task بعداً تکمیل شود یا حالت خاصی (مثلاً تعامل با callbackهای سطح پایین) داشته باشید، از  
`TaskCompletionSource<T>`  
استفاده کنید.

---

### همچنین ببینید

- **دستور ۷.۱:** تست واحد متدهای ناهمگام را پوشش می‌دهد.
- **دستور ۱۱.۱:** ارث‌بری متدهای async را بررسی می‌کند.
- **دستور ۸.۳:** روش استفاده عمومی از `TaskCompletionSource<T>` برای تعامل با سایر کدهای ناهمگام را نشان می‌دهد.

</div>





<div dir="rtl">

## ۲.۳ گزارش‌گیری پیشرفت (Reporting Progress)

### مسئله

هنگام اجرای یک عملیات، لازم دارید به پیشرفت آن واکنش نشان دهید.

---

### راه‌حل

از نوع‌های ارائه‌شده‌ی `IProgress<T>` و `Progress<T>` استفاده کنید.  
متد async شما باید یک پارامتر `IProgress<T>` دریافت کند؛  
`T` هر نوعی است که می‌خواهید پیشرفت را به آن گزارش دهید:

<div dir="ltr" align="left">

```csharp
async Task MyMethodAsync(IProgress<double> progress = null)
{
    bool done = false;
    double percentComplete = 0;
    while (!done)
    {
        ...
        progress?.Report(percentComplete);
    }
}
```

</div>

کد فراخوان به این صورت از آن استفاده می‌کند:

<div dir="ltr" align="left">

```csharp
async Task CallMyMethodAsync()
{
    var progress = new Progress<double>();
    progress.ProgressChanged += (sender, args) =>
    {
        ...
    };
    await MyMethodAsync(progress);
}
```

</div>

---

### بحث

طبق قرارداد، پارامتر `IProgress<T>` می‌تواند `null` باشد اگر فراخواننده نیاز به گزارش پیشرفت ندارد؛  
بنابراین حتماً این پارامتر را در متد async خود بررسی کنید.

توجه داشته باشید که متد `IProgress<T>.Report` معمولاً به صورت ناهمگام اجرا می‌شود.  
یعنی `MyMethodAsync` ممکن است ادامه پیدا کند قبل از اینکه گزارش پیشرفت ارسال یا پردازش شود.  
به همین دلیل، بهتر است `T` را نوعی تغییرناپذیر (immutable) یا دست‌کم یک نوع مقدار (value type) انتخاب کنید.  
اگر `T`، نوع ارجاعی قابل تغییر (mutable reference type) باشد، باید هر بار که Report فراخوانی می‌شود، یک نمونه جدید از آن بسازید.

شیء `Progress<T>` هنگام ساخته‌شدن context جاری را Capture می‌کند و callback خود را در همان context اجرا خواهد کرد.  
به این معنا که اگر `Progress<T>` را روی thread رابط کاربری (UI) بسازید، می‌توانید در callback آن، UI را به‌روز کنید،  
حتی اگر متد async از thread پس‌زمینه، متد `Report` را فراخوانی کند.

وقتی متدی قابلیت گزارش‌گیری پیشرفت دارد، باید تا حد امکان پشتیبانی از لغو (cancellation) را هم فراهم کند.

توجه داشته باشید که `IProgress<T>` مخصوص کد ناهمگام نیست؛  
هم پیشرفت و هم لغو می‌توانند و باید در کدهای همزمان (synchronous) طولانی نیز استفاده شوند.

---

## ۲.۴ منتظر ماندن برای اتمام مجموعه‌ای از وظایف (Tasks)

### مسئله

چندین وظیفه (Task) دارید و نیاز دارید تا همه آن‌ها تمام شوند.

---

### راه‌حل

فریم‌ورک متدی به نام `Task.WhenAll` برای این هدف ارائه می‌دهد.  
این متد تعدادی task دریافت می‌کند و taskی بازمی‌گرداند که وقتی همه آن وظایف کامل شدند، کامل شود:

<div dir="ltr" align="left">

```csharp
Task task1 = Task.Delay(TimeSpan.FromSeconds(1));
Task task2 = Task.Delay(TimeSpan.FromSeconds(2));
Task task3 = Task.Delay(TimeSpan.FromSeconds(1));
await Task.WhenAll(task1, task2, task3);
```

</div>

اگر همه‌ی taskها از یک نوع نتیجه باشند و همه با موفقیت کامل شوند، متد `Task.WhenAll` آرایه‌ای شامل تمام نتایج برمی‌گرداند:

<div dir="ltr" align="left">

```csharp
Task<int> task1 = Task.FromResult(3);
Task<int> task2 = Task.FromResult(5);
Task<int> task3 = Task.FromResult(7);
int[] results = await Task.WhenAll(task1, task2, task3);
// "results" شامل { 3, 5, 7 } است
```

</div>

یک اورلود از `Task.WhenAll` هم وجود دارد که یک `IEnumerable` از taskها را می‌گیرد؛  
اما توصیه نمی‌کنم از آن به طور مستقیم استفاده کنید. هر زمان که کدهای ناهمگام را با LINQ ترکیب می‌کنم،  
متوجه می‌شوم کد زمانی خواناتر است که توالی مورد نظر را (یعنی با ToArray یا متد مشابه) ارزیابی کنم و به صورت کالکشن بنویسم:

<div dir="ltr" align="left">

```csharp
async Task<string> DownloadAllAsync(HttpClient client, IEnumerable<string> urls)
{
    // تعریف اکشن برای هر URL
    var downloads = urls.Select(url => client.GetStringAsync(url));
    // در این مرحله هیچ taskی هنوز شروع نشده چون sequence ارزیابی نشده است.
    // همه دانلودها را همزمان شروع کن.
    Task<string>[] downloadTasks = downloads.ToArray();
    // اکنون همه taskها آغاز شده‌اند.
    // به صورت ناهمگام تا اتمام همه دانلودها صبر کن.
    string[] htmlPages = await Task.WhenAll(downloadTasks);
    return string.Concat(htmlPages);
}
```

</div>

---

### بحث

اگر هر یک از taskها استثنا پرتاب کند، `Task.WhenAll`،  
task بازگشتی خود را با همان استثنا fault می‌کند.  
اگر چندین وظیفه استثنا پرتاب کنند، همه آن استثناها در ویژگی `Exception` از Task برگشتی قرار داده می‌شوند.  
اما هنگام await کردن، تنها یکی از آن‌ها پرتاب خواهد شد.  
اگر شما به تمام استثناها نیاز دارید، می‌توانید ویژگی `Exception` در Task برگشتی از `Task.WhenAll` را بررسی کنید:

<div dir="ltr" align="left">

```csharp
async Task ThrowNotImplementedExceptionAsync()
{
    throw new NotImplementedException();
}
async Task ThrowInvalidOperationExceptionAsync()
{
    throw new InvalidOperationException();
}
async Task ObserveOneExceptionAsync()
{
    var task1 = ThrowNotImplementedExceptionAsync();
    var task2 = ThrowInvalidOperationExceptionAsync();
    try
    {
        await Task.WhenAll(task1, task2);
    }
    catch (Exception ex)
    {
        // ex یا NotImplementedException است یا InvalidOperationException.
        ...
    }
}

async Task ObserveAllExceptionsAsync()
{
    var task1 = ThrowNotImplementedExceptionAsync();
    var task2 = ThrowInvalidOperationExceptionAsync();
    Task allTasks = Task.WhenAll(task1, task2);
    try
    {
        await allTasks;
    }
    catch
    {
        AggregateException allExceptions = allTasks.Exception;
        ...
    }
}
```

</div>

در اغلب اوقات لازم نیست همه استثناها را هنگام استفاده از `Task.WhenAll` بررسی کنید.  
معمولاً واکنش نشان دادن به اولین خطا کافی است.

توجه کنید در مثال بالا، متدهای  
`ThrowNotImplementedExceptionAsync`  
و  
`ThrowInvalidOperationExceptionAsync`  
استثناها را به صورت مستقیم پرتاب نمی‌کنند؛ بلکه چون async هستند،  
استثناها گرفته شده و بر روی task مربوطه قرار می‌گیرند.  
این رفتار معمولی و مورد انتظار متدهایی است که نوع بازگشتی awaitable دارند.

---

### همچنین ببینید

- **دستور ۲.۵:** شیوه‌ای برای انتظار تا پایان «یکی» از مجموعه وظایف را توضیح می‌دهد.
- **دستور ۲.۶:** منتظر ماندن برای پایان مجموعه وظایف و اجرای عملیات بلافاصله پس از هر پایان را پوشش می‌دهد.
- **دستور ۲.۸:** مدیریت استثنا در متدهای async Task را شرح می‌دهد.

</div>


## ۲.۵ منتظر ماندن برای اتمام یکی از وظایف (Tasks)

### مسئله

چندین task دارید و فقط می‌خواهید به اولین task که کامل می‌شود واکنش نشان دهید.  
این مشکل معمولاً زمانی رخ می‌دهد که چندین تلاش مستقل برای انجام یک عملیات دارید، و اولین موفقیت برای شما کافی است.  
به عنوان مثال، می‌توانید همزمان قیمت سهام را از چندین سرویس وب درخواست کنید و فقط به اولین پاسخی که دریافت می‌شود اهمیت دهید.

---

### راه‌حل

از متد `Task.WhenAny` استفاده کنید.  
این متد یک مجموعه از taskها را دریافت می‌کند و وظیفه‌ای برمی‌گرداند که زمانی تکمیل می‌شود که هر کدام از taskها به پایان برسد.  
نتیجه کار این وظیفه بازگشتی، همان taskی است که کامل شده است.  
نمونه کد زیر مفهوم را به شکل دقیق روشن می‌کند:

<div dir="ltr" align="left">

```csharp
// طول داده‌ی اولین URL که پاسخ دهد را بازمی‌گرداند.
async Task<int> FirstRespondingUrlAsync(HttpClient client, string urlA, string urlB)
{
    // هر دو دانلود را به طور همزمان آغاز کن.
    Task<byte[]> downloadTaskA = client.GetByteArrayAsync(urlA);
    Task<byte[]> downloadTaskB = client.GetByteArrayAsync(urlB);

    // منتظر بمان تا یکی از taskها کامل شود.
    Task<byte[]> completedTask = await Task.WhenAny(downloadTaskA, downloadTaskB);

    // طول داده‌ای که از آن URL دریافت کردیم را بازگردان.
    byte[] data = await completedTask;
    return data.Length;
}
```

</div>

<div dir="rtl" align="Right">


### بحث

وظیفه‌ای که توسط `Task.WhenAny` بازمی‌گردد، هرگز در حالت faulted یا canceled تکمیل نمی‌شود.  
این "وظیفه بیرونی" همیشه با موفقیت پایان می‌یابد و مقدار بازگشتی آن همان اولین Task تکمیل‌شده (وظیفه درونی) است.  
اگر task درونی با استثنا کامل شده باشد، آن استثنا به وظیفه بیرونی منتقل نمی‌شود؛  
بلکه شما باید پس از اتمام، آن task را `await` کنید تا مطمئن شوید خطا مشاهده (observed) شده است.

زمانی که اولین task کامل شد، به این مسئله فکر کنید که آیا لازم است taskهای باقی‌مانده را لغو کنید یا نه.  
اگر بقیه taskها لغو نشوند ولی دیگر هرگز await نشوند، این وظایف رهاشده‌اند (abandoned).  
وظایف رهاشده تا پایان ادامه می‌یابند و نتیجه‌شان نادیده گرفته می‌شود و استثناهای آن‌ها نیز نادیده گرفته خواهد شد.  
اگر این taskها لغو نشوند، به مصرف منابع ادامه می‌دهند؛ مانند اتصالات HTTP، پایگاه داده یا تایمرها.

می‌توان از `Task.WhenAny` برای پیاده‌سازی تایم‌اوت (مثلاً با قرار دادن `Task.Delay` به عنوان یکی از taskها) استفاده کرد،  
اما راه مناسب‌تر برای پیاده‌سازی تایم‌اوت، استفاده از cancellation است؛  
ضمن اینکه با این روش واقعاً می‌توان عملیات را در صورت تایم‌اوت لغو کرد.

یکی از ضدالگوهای (anti-pattern) رایج در استفاده از `Task.WhenAny`، استفاده برای بررسی taskها به محض اتمام هرکدام است  
(یعنی فهرستی از taskها نگه دارید و هر کدام که کامل شد حذف کنید).  
این رویکرد زمان اجرا O(N²) دارد؛ در حالی که برای این مسئله الگوریتمی با زمان اجرای O(N) وجود دارد.  
روش صحیح را در دستور ۲.۶ بخوانید.

---

### همچنین ببینید

- **دستور ۲.۴:** شیوه اجرای asynchronous برای صبرکردن تا تکمیل همه taskها را توضیح می‌دهد.
- **دستور ۲.۶:** صبر برای اتمام یک مجموعه task و انجام عملیات پس از اتمام هر کدام را شرح می‌دهد.
- **دستور ۱۰.۳:** استفاده از CancellationToken برای پیاده‌سازی تایم‌اوت را پوشش می‌دهد.

</div>


</div>




<div dir="rtl">

## ۲.۶ پردازش وظایف هنگام کامل شدن هرکدام (Processing Tasks as They Complete)

### مسئله

یک مجموعه از وظایف (Tasks) دارید و می‌خواهید بلافاصله پس از کامل شدن هر کدام، عملیاتی روی آن انجام دهید؛ بدون اینکه منتظر تکمیل بقیه بمانید.

در مثال زیر سه task تاخیری آغاز می‌شود و سپس هر کدام به‌ترتیب در لیست، `await` و پردازش می‌شود:

<div dir="ltr">

```csharp
async Task<int> DelayAndReturnAsync(int value)
{
    await Task.Delay(TimeSpan.FromSeconds(value));
    return value;
}

// فعلاً این متد "۲"، "۳" و "۱" را به همین ترتیب چاپ می‌کند.
// اما رفتار مطلوب این است که "۱"، "۲" و "۳" به‌ترتیب کامل‌شدن چاپ شود.
async Task ProcessTasksAsync()
{
    // مجموعه‌ای از taskها بسازید.
    Task<int> taskA = DelayAndReturnAsync(2);
    Task<int> taskB = DelayAndReturnAsync(3);
    Task<int> taskC = DelayAndReturnAsync(1);
    Task<int>[] tasks = new[] { taskA, taskB, taskC };

    // هر task را به‌ترتیب سری (نه ترتیب تکمیل) await کن.
    foreach (Task<int> task in tasks)
    {
        var result = await task;
        Trace.WriteLine(result);
    }
}
```

</div>

در کد بالا، هر task به ترتیب ابتدایی اجرا ذخیره شده و به‌ترتیب لیست پردازش می‌شود، حتی اگر task سوم زودتر تمام شود. هدف این است که پردازش بلافاصله پس از کامل‌شدن هر task و نه به‌ترتیب لیست، انجام شود.

---

### راه‌حل

روش‌های مختلفی وجود دارد، اما بهترین روش پیشنهادشده این دستور، بازآرایی (refactor) کد است تا هر task به‌محض کامل‌شدن پردازش شود. با این کار کد خواناتر و مدرن‌تر خواهد بود:

<div dir="ltr">

```csharp
async Task<int> DelayAndReturnAsync(int value)
{
    await Task.Delay(TimeSpan.FromSeconds(value));
    return value;
}

async Task AwaitAndProcessAsync(Task<int> task)
{
    int result = await task;
    Trace.WriteLine(result);
}

// حالا این متد خروجی را به‌ترتیب "۱"، "۲" و "۳" چاپ می‌کند.
async Task ProcessTasksAsync()
{
    // مجموعه‌ای از taskها بسازید.
    Task<int> taskA = DelayAndReturnAsync(2);
    Task<int> taskB = DelayAndReturnAsync(3);
    Task<int> taskC = DelayAndReturnAsync(1);
    Task<int>[] tasks = new[] { taskA, taskB, taskC };

    IEnumerable<Task> taskQuery = from t in tasks select AwaitAndProcessAsync(t);
    Task[] processingTasks = taskQuery.ToArray();
    // منتظر بمان همه پردازش‌ها کامل شوند.
    await Task.WhenAll(processingTasks);
}
```

</div>

**روش جایگزین (با بازنویسی کمتر):**

<div dir="ltr">

```csharp
async Task<int> DelayAndReturnAsync(int value)
{
    await Task.Delay(TimeSpan.FromSeconds(value));
    return value;
}

// باز هم خروجی "۱"، "۲" و "۳" خواهد بود.
async Task ProcessTasksAsync()
{
    Task<int> taskA = DelayAndReturnAsync(2);
    Task<int> taskB = DelayAndReturnAsync(3);
    Task<int> taskC = DelayAndReturnAsync(1);
    Task<int>[] tasks = new[] { taskA, taskB, taskC };

    Task[] processingTasks = tasks.Select(async t =>
    {
        var result = await t;
        Trace.WriteLine(result);
    }).ToArray();
    await Task.WhenAll(processingTasks);
}
```

</div>

این مدل بازآرایی‌شده بهترین و تمیزترین راه برای حل این مسأله است.  
توجه کنید که این راه‌حل نسبت به نسخه اولیه این تفاوت را دارد که عملیات پردازشی همزمان با هم (concurrent) اجرا می‌شوند،  
ولی در کد نخست فقط یکی یکی و به‌ترتیب اجرا می‌شد.  
اگر این رفتار همزمان مناسب نیست، می‌توانید از قفل (lock) استفاده کنید (دستور ۱۲.۲)، یا راه جایگزین زیر را به کار بگیرید.

---

### بحث

اگر بازآرایی (refactoring) ممکن یا مطلوب نیست، جایگزینی وجود دارد: استفن توب و جان اسکیت هر دو متدی الحاقی (extension method) معرفی کرده‌اند که یک آرایه از taskها را به‌ترتیب تکمیل‌شان بازمی‌گرداند.

روش استفن توب در بلاگ "Parallel Programming with .NET" و روش جان اسکیت در بلاگ شخصی‌اش آمده‌است.

**نکته:**  
متد `OrderByCompletion` به صورت open source در کتابخانه AsyncEx (و بسته NuGet با نام Nito.AsyncEx) هم موجود است.

</div>






<div dir="rtl">

## استفاده از متد الحاقی OrderByCompletion

با این extension method، تغییر زیادی نیاز ندارید:

<div dir="ltr">

```csharp
async Task<int> DelayAndReturnAsync(int value)
{
    await Task.Delay(TimeSpan.FromSeconds(value));
    return value;
}

// حالا این متد هم "۱"، "۲" و "۳" را به‌ترتیب تکمیل چاپ می‌کند.
async Task UseOrderByCompletionAsync()
{
    Task<int> taskA = DelayAndReturnAsync(2);
    Task<int> taskB = DelayAndReturnAsync(3);
    Task<int> taskC = DelayAndReturnAsync(1);
    Task<int>[] tasks = new[] { taskA, taskB, taskC };

    // هر کدام که کامل شد، همین که آماده شد await و پردازش کن:
    foreach (Task<int> task in tasks.OrderByCompletion())
    {
        int result = await task;
        Trace.WriteLine(result);
    }
}
```
</div>

---

### همچنین ببینید

- **دستور ۲.۴:** صبر کردن برای کامل شدن همه وظایف به‌شکل ناهمگام را شرح می‌دهد.

---

## ۲.۷ جلوگیری از بازگشت ادامه (Continuation) به Context

### مسئله

وقتی یک متد async پس از یک await ادامه پیدا می‌کند، به طور پیش‌فرض، کار را در همان context (مثلاً context رابط کاربری) از سر می‌گیرد.  
این موضوع در صورتی که context یک UI context باشد و تعداد زیادی متد async همزمان روی آن resume شوند، می‌تواند باعث مشکلات کارایی (Performance) شود.

---

### راه‌حل

برای جلوگیری از بازگشت continuation به context اولیه، کافی است موقع await کردن از  
`ConfigureAwait(false)`  
استفاده کنید:

<div dir="ltr">

```csharp
async Task ResumeOnContextAsync()
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    // این متد پس از اتمام await، در همان context (مثلاً UI thread) ادامه می‌یابد.
}

async Task ResumeWithoutContextAsync()
{
    await Task.Delay(TimeSpan.FromSeconds(1)).ConfigureAwait(false);
    // این متد پس از اتمام await، context را رها می‌کند.
}
```
</div>

---

### بحث

اگر تعداد زیادی continuation روی thread رابط کاربری اجرا شود، ممکن است افت کارایی شدیدی رخ دهد.  
این نوع مشکلات کارایی، به دلیل اینکه ناشی از یک متد خاص نیستند بلکه جمع بسیار زیادی از اجرای متدها هستند، تشخیص‌شان سخت است؛  
اغلب باعث می‌شوند UI برنامه به‌تدریج کند شود و احساس کاربر نسبت به عملکرد ضعیف شود.

اما واقعاً چند تا continuation روی UI thread زیاد محسوب می‌شود؟  
پاسخ قطعی وجود ندارد، اما لوسیان ویشیک از مایکروسافت راهنمای تیم Universal Windows را این طور توضیح داده:  
حدود صد تای آن در هر ثانیه خوب است، اما هزار تای آن در ثانیه بیش از حد است و باید اجتناب شود.

بنابراین، برای پیشگیری از این مشکل بهتر است از ابتدا آگاه باشید و عادت کنید هر زمان که یک متد async  
نیاز ندارد به context اصلی بازگردد، از  
`ConfigureAwait(false)`  
استفاده کنید. این کار هیچ ضرری ندارد!

همچنین خوب است هنگام نوشتن کد async کاملاً نسبت به context آگاه باشید.  
یک متد async معمولاً باید یا کاملاً به context نیاز داشته باشد (مثلاً تعامل با UI یا درخواست/پاسخ‌های ASP.NET)  
یا کاملاً بی‌نیاز از context باشد (مثلاً عملیات پس‌زمینه).  
اگر متدی دارید که بخش‌هایی از آن نیازمند context و بخش‌هایی غیر وابسته‌اند، آن را به متدهای کوچکتر (دو یا چند متد async)  
تقسیم کنید؛ این کار باعث سازمان‌دهی بهتر کد و لایه‌بندی صحیح‌تر می‌شود.

</div>





<div dir="rtl">

---

### همچنین ببینید

- **فصل ۱:** مقدمه‌ای بر برنامه‌نویسی ناهمگام را ارائه می‌دهد.

---

## ۲.۸ مدیریت استثناها در متدهای async Task

### مسئله

مدیریت استثنا بخشی حیاتی از طراحی هر برنامه است. طراحی برای حالت موفقیت آسان است؛  
اما تا زمانی که شکست‌ها هم مدیریت نشوند، آن طراحی کامل نیست. خوشبختانه مدیریت استثناها در متدهای async Task ساده و سرراست است.

---

### راه‌حل

برای گرفتن استثناها کافی است مانند کد همزمان از دستور try/catch استفاده کنید:

<div dir="ltr">

```csharp
async Task ThrowExceptionAsync()
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    throw new InvalidOperationException("Test");
}

async Task TestAsync()
{
    try
    {
        await ThrowExceptionAsync();
    }
    catch (InvalidOperationException)
    {
        // استثنا به درستی گرفته می‌شود.
    }
}
```
</div>

استثناهای پرتاب شده از متدهای async Task روی Task برگشتی قرار می‌گیرند.  
این استثناها فقط زمانی پرتاب می‌شوند که آن Task را await کنید:

<div dir="ltr">

```csharp
async Task ThrowExceptionAsync()
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    throw new InvalidOperationException("Test");
}

async Task TestAsync()
{
    // استثنا توسط متد پرتاب می‌شود و روی Task قرار می‌گیرد.
    Task task = ThrowExceptionAsync();
    try
    {
        // استثنا دقیقاً اینجا، هنگام await کردن task، دوباره پرتاب می‌شود.
        await task;
    }
    catch (InvalidOperationException)
    {
        // و اینجا به درستی گرفته می‌شود.
    }
}
```
</div>

---

### بحث

وقتی یک استثنا از یک متد async Task پرتاب می‌شود، آن استثنا گرفته شده و روی Task برگشتی ذخیره می‌شود.  
متدهای async void این رفتار را ندارند (چون Taskی برای ذخیره استثنا ندارند) و مدیریت استثنا در آنها متفاوت است (به دستور ۲.۹ مراجعه کنید).

هنگامی که یک Task fault شده را await می‌کنید، اولین استثنای روی آن Task دوباره پرتاب می‌شود.  
ممکن است درباره وضعیت stack trace تردید داشته باشید؛ اما مطمئن باشید که هنگام پرتاب مجدد، stack trace اصلی به‌درستی حفظ می‌شود.

این سازوکار شاید پیچیده به نظر برسد، اما همه این پیچیدگی‌ها برای آن است که در سناریوهای معمول، کد شما ساده و قابل درک باقی بماند.  
در اغلب موارد، کافیست متد async را await کنید و اگر خطایی رخ داد، استثنای آن توسط try/catch مدیریت خواهد شد.

در برخی شرایط (مثل Task.WhenAll)، ممکن است یک Task چندین استثنا داشته باشد ولی await فقط اولین مورد را دوباره پرتاب می‌کند.  
برای مدیریت همه استثناها به دستور ۲.۴ رجوع کنید.

---

### همچنین ببینید

- **دستور ۲.۴:** انتظار برای کامل شدن چندین task را شرح می‌دهد.
- **دستور ۲.۹:** روش‌های گرفتن استثنا از متدهای async void را توضیح می‌دهد.
- **دستور ۷.۲:** تست واحد استثناهای پرتاب‌شده از async Task را پوشش می‌دهد.

---

## ۲.۹ مدیریت استثناها در متدهای async void

### مسئله

یک متد async void دارید و باید استثناهایی که از آن پراپاگیت می‌شوند را مدیریت کنید.

---

### راه‌حل

واقعیت این است که هیچ راه حل ایده‌آلی وجود ندارد.  
اگر امکان‌پذیر است، متد خود را طوری تغییر دهید که به جای void، مقدار Task بازگرداند.

البته گاهی این کار ممکن نیست؛  
مثلاً وقتی می‌خواهید یک پیاده‌سازی از ICommand (.NET) بنویسید که بر اساس قرارداد باید void برگرداند.  
در این حالت، می‌توانید یک overload از متد اجرا داشته باشید که Task برمی‌گرداند:

<div dir="ltr">

```csharp
sealed class MyAsyncCommand : ICommand
{
    async void ICommand.Execute(object parameter)
    {
        await Execute(parameter);
    }

    public async Task Execute(object parameter)
    {
        // کد اصلی ناهمگام مربوط به دستور اینجاست
        ...
    }

    // بقیه اعضا (مثل CanExecute و غیره)
}
```
</div>

بهترین کار این است که اجازه ندهید استثناها از متدهای async void خارج شوند.  
اگر مجبور به استفاده از async void هستید، بهتر است همه کد مربوط به آن را در یک بلاک try قرار دهید و استثناها را همانجا مدیریت کنید.

</div>



<div dir="rtl">

---

### راه‌ دیگری برای مدیریت استثناهایی که از async void پراپاگیت می‌شوند

وقتی یک متد async void یک استثنا تولید می‌کند، آن استثنا روی SynchronizationContext فعالی که هنگام شروع متد وجود داشته برقرار می‌شود.  
اگر محیط اجرایی شما SynchronizationContext داشته باشد، معمولاً راهی برای مدیریت اینگونه استثناها در سطح سراسری برنامه وجود دارد.

**برای نمونه:**

- در **WPF**: استفاده از رویداد `Application.DispatcherUnhandledException`
- در **Universal Windows**: استفاده از رویداد `Application.UnhandledException`
- در **ASP.NET**: استفاده از Middleware با نام `UseExceptionHandler`

راه دیگر کنترل و مدیریت این استثناها، مدیریت مستقیم SynchronizationContext است.  
نوشتن Context خودتان ساده نیست، ولی می‌توانید از **AsyncContext** در کتابخانه رایگان Nito.AsyncEx استفاده کنید.

این Context به‌ویژه برای برنامه‌هایی مثل Console app یا سرویس‌های Win32 مفید است که به صورت پیش‌فرض SynchronizationContext ندارند.

مثال زیر نحوه استفاده از AsyncContext برای مدیریت استثناهای متد async void را نشان می‌دهد:

<div dir="ltr">

```csharp
static class Program
{
    static void Main(string[] args)
    {
        try
        {
            AsyncContext.Run(() => MainAsync(args));
        }
        catch (Exception ex)
        {
            Console.Error.WriteLine(ex);
        }
    }

    // هشدار: در دنیای واقعی، فقط اگر مجبورید از async void استفاده کنید!
    static async void MainAsync(string[] args)
    {
        ...
    }
}
```
</div>

---

### بحث

یکی از دلایل مهم برای ترجیح دادن متدهای async Task نسبت به async void، سهولت تست‌نویسی است.  
حتی اگر مجبور به داشتن متد void هستید، اضافه‌کردن overload با Task باعث می‌شود سطح API شما تست‌پذیر بماند.

اگر نیاز به ساخت SynchronizationContext اختصاصی خودتان پیدا کردید (مثل AsyncContext)، دقت کنید که آن را فقط روی threadهایی نصب کنید که متعلق به خودتان است.  
به طور کلی، هرگز این context را روی threadهایی که از پیش یک SynchronizationContext دارند (مثلاً UI thread یا ASP.NET request thread) یا روی threadpool ها نصب نکنید.  
Thread اصلی برنامه‌های Console و threadهای خلق‌شده توسط خودتان، مکان‌های مجاز نصب هستند.

**نکته:**  
نوع `AsyncContext` در بسته NuGet به نام **Nito.AsyncEx** موجود است.

---

### همچنین ببینید

- **دستور ۲.۸:** مدیریت استثنا برای متدهای async Task را پوشش می‌دهد.
- **دستور ۷.۳:** تست واحد متدهای async void را شرح می‌دهد.

---

## ۲.۱۰ ایجاد (ساختن) ValueTask

### مسئله

نیاز دارید متدی پیاده‌سازی کنید که خروجی آن `ValueTask<T>` باشد.

---

### راه‌حل

نوع `ValueTask<T>` زمانی به عنوان خروجی متد به کار می‌رود که معمولاً می‌توان نتیجه را به صورت همزمان (sync) بازگرداند و رفتار ناهمگام (async) تنها در موارد کمی رخ می‌دهد.

در کدنویسی معمولی، **ترجیحاً از Task<T> به جای ValueTask<T> استفاده کنید**  
(مگر این که با بررسی پروفایل و کارایی مطمئن شوید بهبود قابل توجهی خواهید داشت).

با این حال، گاهی لازم است متدی را پیاده کنید که خروجی `ValueTask<T>` داشته باشد. مثلاً در پیاده‌سازی `IAsyncDisposable`، که متد `DisposeAsync` آن از نوع `ValueTask` است.

ساده‌ترین روش: درست مثل متد async معمولی از async و await استفاده کنید:

<div dir="ltr">

```csharp
public async ValueTask<int> MethodAsync()
{
    await Task.Delay(100); // کار ناهمگام...
    return 13;
}
```
</div>

بسیاری موارد، متد ValueTask می‌تواند فوراً مقدار را بازگرداند؛ فقط اگر نیاز بود async را صدا می‌زنید:

<div dir="ltr">

```csharp
public ValueTask<int> MethodAsync()
{
    if (CanBehaveSynchronously)
        return new ValueTask<int>(13);
    return new ValueTask<int>(SlowMethodAsync());
}

private Task<int> SlowMethodAsync();
```
</div>

رویکرد مشابهی برای ValueTask غیربازپرداختی (غیر generic) هم وجود دارد.  
در این حالت می‌توانید از سازنده پیش‌فرض ValueTask برای بازگرداندن یک ValueTask به صورت موفق و سریع استفاده کنید.

مثال زیر، یک پیاده‌سازی از IAsyncDisposable را نشان می‌دهد که منطق پاکسازی ناهمگام فقط یک‌بار اجرا می‌شود و در دفعات بعدی،  
DisposeAsync فوراً و sync به پایان می‌رسد:

<div dir="ltr">

```csharp
private Func<Task> _disposeLogic;

public ValueTask DisposeAsync()
{
    if (_disposeLogic == null)
        return default;
    // توجه: این مثال ساده ایمن نسبت به thread نیست؛ اگر چند thread 
    // بصورت همزمان DisposeAsync را صدا بزنند، منطق پاکسازی ممکن است چندبار اجرا شود.
    Func<Task> logic = _disposeLogic;
    _disposeLogic = null;
    return new ValueTask(logic());
}
```
</div>

---

### بحث

در اغلب مواقع، متدهای شما باید Task<T> بازگردانند؛  
چرا که مصرف Task<T> آسان‌تر و بدون دردسرتر از ValueTask<T> است. جزئیات این محدودیت‌ها در دستور ۲.۱۱ توضیح داده شده است.

اکثر اوقات، اگر فقط اینترفیس‌هایی را پیاده می‌کنید که ValueTask یا ValueTask<T> دارند، با async/await به راحتی می‌توانید بنویسید.  
موارد پیشرفته‌تر زمانی پیش می‌آید که خودتان بخواهید از ValueTask<T> استفاده کنید.

روش‌های معمول و ساده در همین دستور توضیح داده شده است، اما راه‌های پیچیده‌تری هم هست (مثلاً پیاده‌سازی  
IValueTaskSource<T> برای سناریوهای بهینه‌سازی حافظه و کش).  
برای این‌ها، به مستندات مایکروسافت درباره‌ی  
`ManualResetValueTaskSourceCore<T>`  
مراجعه کنید.

---

### همچنین ببینید

- **دستور ۲.۱۱:** محدودیت‌های مصرف ValueTask<T> و ValueTask را شرح می‌دهد.
- **دستور ۱۱.۶:** پاکسازی ناهمگام را پوشش می‌دهد.

</div>




<div dir="rtl">

## ۲.۱۱ مصرف (استفاده از) ValueTask

### مسئله

نیاز دارید یک مقدار ValueTask<T> را مصرف (await) کنید.

---

### راه‌حل

ساده‌ترین و رایج‌ترین روش مصرف کردن یک مقدار ValueTask<T> (یا ValueTask)، استفاده از await است. معمولاً همین رجوع مستقیم کافی است:

<div dir="ltr">

```csharp
ValueTask<int> MethodAsync();

async Task ConsumingMethodAsync()
{
    int value = await MethodAsync();
}
```
</div>

همچنین می‌توانید ابتدا ValueTask را بگیرید و پس از انجام عملیات‌های موازی دیگر، آن را await کنید (عین Task<T>):

<div dir="ltr">

```csharp
ValueTask<int> MethodAsync();

async Task ConsumingMethodAsync()
{
    ValueTask<int> valueTask = MethodAsync();
    ... // انجام کارهای همزمان دیگر
    int value = await valueTask;
}
```
</div>

هر دوی این روش‌ها مناسب‌اند، زیرا مقدار ValueTask فقط یک بار await می‌شود.  
این محدودیتی کلیدی در استفاده از ValueTask است.

> **هشدار:**  
> یک ValueTask یا ValueTask<T> فقط باید یک بار await شود.

اگر نیاز به کارهای پیچیده‌تری دارید (مثلاً چند بار await کردن، اشتراک‌گذاری، یا موارد مشابه)، مقدار ValueTask<T> را با متد `AsTask` به یک Task<T> تبدیل کنید:

<div dir="ltr">

```csharp
ValueTask<int> MethodAsync();

async Task ConsumingMethodAsync()
{
    Task<int> task = MethodAsync().AsTask();
    ... // کارهای موازی دیگر
    int value = await task;
    int anotherValue = await task;   // مشکلی نیست
}
```
</div>

await کردن یک Task<T> چندین بار کاملاً امن است و می‌توانید کارهای دیگر مثل صبر برای تکمیل چند وظیفه (مثل دستور ۲.۴) را هم انجام دهید:

<div dir="ltr">

```csharp
ValueTask<int> MethodAsync();

async Task ConsumingMethodAsync()
{
    Task<int> task1 = MethodAsync().AsTask();
    Task<int> task2 = MethodAsync().AsTask();
    int[] results = await Task.WhenAll(task1, task2);
}
```
</div>

اما توجه کنید: برای هر ValueTask<T> فقط یک بار می‌توانید AsTask را صدا بزنید.  
معمولاً بهتر است فوراً آن را به Task<T> تبدیل کنید و سپس دیگر سراغ ValueTask<T> نروید.  
همچنین نمی‌توانید یک ValueTask<T> را هم await کنید و هم روی همان مقدار AsTask بزنید.

پس، غالباً کد شما یا باید ValueTask<T> را بلافاصله await کند یا بلافاصله به Task<T> تبدیل و از آن به بعد با Task کار کند.

---

### بحث

سایر پراپرتی‌های ValueTask<T> عمدتاً برای استفاده‌های پیشرفته‌تر است و رفتاری شبیه پراپرتی‌های Task<T> ندارد؛  
مثلاً ValueTask<T>.Result محدودیت بیشتری نسبت به Task<T>.Result دارد.

اگر می‌خواهید نتیجه ValueTask<T> را به‌صورت همزمان (سینکرون) دریافت کنید، می‌توانید از `.Result` یا  
`GetAwaiter().GetResult()`  
استفاده کنید، اما این کار فقط زمانی مجاز است که ValueTask<T> حتماً کامل شده باشد.

همچنین برخلاف Task<T> که وقتی `.Result` را می‌زنید تا کامل شدن عملیات thread فعلی را معطل می‌کند، ValueTask<T> چنین ضمانتی ندارد و ممکن است کار نکند یا اشکال ایجاد کند.

> **هشدار:**  
> گرفتن نتیجه سینکرون از ValueTask یا ValueTask<T> فقط یک بار، و فقط پس از کامل شدن، مجاز است و همان valueTask دیگر نباید await یا تبدیل به Task شود.

خلاصه و تاکید:  
وقتی متدی را صدا می‌کنید که ValueTask یا ValueTask<T> برمی‌گرداند، یا سریع آن را await کنید یا سریعاً به Task تبدیل و از Task استفاده کنید.  
این قاعده در اکثر سناریوهای عادی کافی است و موارد پیشرفته‌تر چندان رایج نیست.

---

### همچنین ببینید

- **دستور ۲.۱۰:** نحوه بازگرداندن ValueTask<T> و ValueTask را از متدها شرح می‌دهد.
- **دستورهای ۲.۴ و ۲.۵:** صبر برای چندین Task به صورت همزمان را پوشش می‌دهند.

</div>
