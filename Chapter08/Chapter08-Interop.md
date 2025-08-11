
<div dir="rtl">

# فصل ۸. درهم‌کنش (Interop)

ناهمگام، موازی، واکنشی — هرکدام جایگاه خودشان را دارند، اما چقدر خوب با هم کار می‌کنند؟

در این فصل، به سناریوهای مختلف درهم‌کنش می‌پردازیم تا یاد بگیرید چگونه این رویکردهای متفاوت را با هم ترکیب کنید. خواهید دید که آن‌ها مکمل هم هستند نه رقیب؛ اصطکاک بسیار کمی در مرزهایی وجود دارد که یک رویکرد به رویکرد دیگر می‌رسد.

---

## 8.1. پوشاننده‌های Async برای متدهای «Async» با رویدادهای «Completed»

### مسئله
یک الگوی ناهمگام قدیمی وجود دارد که از متدهایی با نام `OperationAsync` همراه با رویدادهایی با نام `OperationCompleted` استفاده می‌کند. می‌خواهید عملیاتی را با استفاده از این الگوی قدیمی انجام دهید و نتیجه را `await` کنید.

### نکته
الگوی `OperationAsync` و `OperationCompleted` «الگوی ناهمگام مبتنی بر رویداد» (Event-based Asynchronous Pattern یا EAP) نام دارد. قصد دارید آن‌ها را در متدی که `Task` برمی‌گرداند و از «الگوی ناهمگام مبتنی بر Task» (Task-based Asynchronous Pattern یا TAP) پیروی می‌کند، بپیچید.

### راه‌حل
با استفاده از نوع `TaskCompletionSource<TResult>` می‌توانید برای عملیات‌های ناهمگام پوشاننده (wrapper) بسازید. نوع `TaskCompletionSource<TResult>` یک `Task<TResult>` را کنترل می‌کند و به شما امکان می‌دهد در زمان مناسب آن Task را کامل کنید.

این مثال یک متد اکستنشن برای `WebClient` تعریف می‌کند که یک رشته را دانلود می‌کند. نوع `WebClient` متدهای `DownloadStringAsync` و رویداد `DownloadStringCompleted` را تعریف کرده است. با استفاده از آن‌ها، می‌توانید متد `DownloadStringTaskAsync` را به شکل زیر تعریف کنید:

</div>

```csharp
public static Task<string> DownloadStringTaskAsync(
    this WebClient client,
    Uri address)
{
    var tcs = new TaskCompletionSource<string>();

    // هندلر رویداد، تسک را کامل کرده و خودش را لغو ثبت می‌کند.
    DownloadStringCompletedEventHandler? handler = null;
    handler = (_, e) =>
    {
        client.DownloadStringCompleted -= handler;

        if (e.Cancelled)
            tcs.TrySetCanceled();
        else if (e.Error != null)
            tcs.TrySetException(e.Error);
        else
            tcs.TrySetResult(e.Result);
    };

    // ابتدا برای رویداد ثبت‌نام کنید و سپس عملیات را شروع کنید.
    client.DownloadStringCompleted += handler;
    client.DownloadStringAsync(address);

    return tcs.Task;
}
```

<div dir="rtl">

### بحث
این مثال مشخصاً خیلی مفید نیست، چون `WebClient` همین حالا هم `DownloadStringTaskAsync` را تعریف کرده و همچنین یک `HttpClient` سازگارتر با async وجود دارد که می‌توان از آن استفاده کرد. با این حال، همین تکنیک را می‌توان برای تعامل با کدهای ناهمگام قدیمی که هنوز به `Task` به‌روزرسانی نشده‌اند به کار برد.

### نکته
 برای کد جدید همیشه از `HttpClient` استفاده کنید. فقط زمانی از `WebClient` استفاده کنید که با کد میراثی (legacy) کار می‌کنید.
 به‌طور معمول، یک متد TAP برای دانلود رشته‌ها به شکل `OperationAsync` (مثلاً `DownloadStringAsync`) نام‌گذاری می‌شود؛ اما در این مورد، این قرارداد نام‌گذاری جواب نمی‌دهد چون EAP همین حالا متدی با آن نام دارد. اینجا قرارداد این است که متد TAP را `OperationTaskAsync` (مثلاً `DownloadStringTaskAsync`) نام‌گذاری کنیم.
 هنگام پیچیدن متدهای EAP، این امکان وجود دارد که متد «شروع» استثنا پرتاب کند؛ در مثال قبلی، `DownloadStringAsync` ممکن است استثنا پرتاب کند. در این صورت باید تصمیم بگیرید که اجازه دهید استثنا منتشر شود یا آن را گرفته و `TrySetException` را فراخوانی کنید. اغلب اوقات، استثناهای پرتاب‌شده در آن نقطه خطاهای نحوه‌استفاده (usage errors) هستند؛ اگر مطمئن نیستید که استثناها از این نوع‌اند، پیشنهاد می‌شود استثنا را بگیرید و `TrySetException` را فراخوانی کنید.

برای مطالعه بیشتر:
 دستورالعمل 8.2 پوشاننده‌های TAP برای متدهای APM (`BeginOperation` و `EndOperation`) را پوشش می‌دهد.
 دستورالعمل 8.3 پوشاننده‌های TAP برای هر نوع اعلان (notification) را پوشش می‌دهد.

---

## 8.2. پوشاننده‌های Async برای متدهای «Begin/End»

### مسئله
یک الگوی ناهمگام قدیمی از جفت متدهایی با نام `BeginOperation` و `EndOperation` استفاده می‌کند که `IAsyncResult` نماینده عملیات ناهمگام است. شما عملیاتی دارید که از این الگوی قدیمی پیروی می‌کند و می‌خواهید آن را با `await` مصرف کنید.

### نکته
الگوی `BeginOperation` و `EndOperation` «مدل برنامه‌نویسی ناهمگام» (Asynchronous Programming Model یا APM) نام دارد. قصد دارید آن‌ها را در متدی که `Task` برمی‌گرداند و از TAP پیروی می‌کند، بپیچید.

### راه‌حل
بهترین روش برای پیچیدن APM استفاده از یکی از متدهای `FromAsync` روی نوع `TaskFactory` است. `FromAsync` زیر پوست خود از `TaskCompletionSource<TResult>` استفاده می‌کند، اما هنگام پیچیدن APM، استفاده از `FromAsync` بسیار آسان‌تر است.

این مثال یک متد اکستنشن برای `WebRequest` تعریف می‌کند که یک درخواست HTTP ارسال کرده و پاسخ آن را دریافت می‌کند. نوع `WebRequest` متدهای `BeginGetResponse` و `EndGetResponse` را تعریف می‌کند؛ می‌توانید متد `GetResponseAsync` را به شکل زیر تعریف کنید:

</div>

```csharp
public static Task<WebResponse> GetResponseAsync(this WebRequest client)
{
    return Task<WebResponse>.Factory.FromAsync(
        client.BeginGetResponse,
        client.EndGetResponse,
        null
    );
}
```

<div dir="rtl">

### بحث
`FromAsync` تعداد وحشت‌آوری سربارگذاری (overload) دارد و گیج‌کننده است!

 به‌طور کلی، بهترین کار این است که `FromAsync` را دقیقاً مانند مثال صدا بزنید. ابتدا متد `BeginOperation` را پاس دهید (بدون اینکه آن را فراخوانی کنید)، سپس متد `EndOperation` را پاس دهید (بدون اینکه آن را فراخوانی کنید). بعد، تمام آرگومان‌هایی را پاس دهید که `BeginOperation` می‌گیرد به‌جز دو آرگومان آخر یعنی `AsyncCallback` و `object`. در نهایت، مقدار `null` را پاس دهید.
 به‌طور خاص، متد `BeginOperation` را قبل از فراخوانی `FromAsync` صدا نزنید. می‌توانید `FromAsync` را به‌نحوی صدا بزنید که `IAsyncResult` بازگشتی از `BeginOperation` را پاس می‌دهد، اما اگر این‌طور صدا بزنید، `FromAsync` مجبور می‌شود از یک پیاده‌سازی کم‌بازده‌تر استفاده کند.
 شاید بپرسید چرا الگوی پیشنهادی همیشه در انتها `null` پاس می‌دهد. `FromAsync` هم‌زمان با نوع `Task` در .NET 4.0 معرفی شد، قبل از اینکه `async` رایج شود. در آن زمان، استفاده از شیء وضعیت (state object) در کال‌بک‌های ناهمگام معمول بود و نوع `Task` از این طریق از طریق عضو `AsyncState` پشتیبانی می‌کند. در الگوی جدید `async` دیگر شیء وضعیت لازم نیست، بنابراین طبیعی است که همیشه برای پارامتر `state` مقدار `null` پاس داده شود. این روزها، `state` فقط برای اجتناب از ساخت closure هنگام بهینه‌سازی مصرف حافظه استفاده می‌شود.

برای مطالعه بیشتر:
 دستورالعمل 8.3 نگارش پوشاننده‌های TAP برای هر نوع اعلان را پوشش می‌دهد.

---

## 8.3. پوشاننده‌های Async برای هر چیزی

### مسئله
یک عملیات یا رویداد ناهمگام غیرمعمول یا غیرمعیار دارید و می‌خواهید آن را از طریق `await` مصرف کنید.

### راه‌حل
نوع `TaskCompletionSource<T>` می‌تواند برای ساختن شیءهای `Task<T>` در هر سناریویی استفاده شود. با استفاده از `TaskCompletionSource<T>` می‌توانید یک تسک را به سه شیوه کامل کنید: با نتیجه موفق، با خطا (faulted)، یا لغوشده.

قبل از اینکه `async` رایج شود، دو الگوی ناهمگام دیگر توسط مایکروسافت توصیه می‌شد: APM (دستورالعمل 8.2) و EAP (دستورالعمل 8.1). با این حال، هر دو APM و EAP تا حدی دست‌وپاگیر بودند و در برخی موارد درست انجام دادن آن‌ها دشوار بود. بنابراین یک قرارداد غیررسمی پدید آمد که از کال‌بک‌ها استفاده می‌کرد، با متدهایی شبیه نمونه زیر:

</div>

```csharp
public interface IMyAsyncHttpService
{
    void DownloadString(Uri address, Action<string, Exception?> callback);
}
```

<div dir="rtl">

متدهایی مانند این از قراردادی پیروی می‌کنند که `DownloadString` دانلود (ناهمگام) را شروع می‌کند و وقتی کامل شد، کال‌بک با نتیجه یا استثنا فراخوانی می‌شود. معمولاً کال‌بک روی یک نخ پس‌زمینه فراخوانی می‌شود.

یک متد ناهمگام غیرمعیاری مانند مثال قبلی را می‌توان با استفاده از `TaskCompletionSource<T>` پیچید تا به‌صورت طبیعی با `await` کار کند، همان‌طور که مثال بعدی نشان می‌دهد:

</div>

```csharp
public static Task<string> DownloadStringAsync(
    this IMyAsyncHttpService httpService,
    Uri address)
{
    var tcs = new TaskCompletionSource<string>();

    httpService.DownloadString(address, (result, exception) =>
    {
        if (exception != null)
            tcs.TrySetException(exception);
        else
            tcs.TrySetResult(result);
    });

    return tcs.Task;
}
```

<div dir="rtl">

### بحث
می‌توانید همین الگوی `TaskCompletionSource<T>` را برای پیچیدن هر متد ناهمگامی به کار ببرید، فارغ از اینکه چقدر غیرمعیار باشد. ابتدا نمونه `TaskCompletionSource<T>` را ایجاد کنید. سپس ترتیبی دهید که یک کال‌بک طوری تنظیم شود که `TaskCompletionSource<T>` تسک وابسته‌اش را به‌صورت مناسب کامل کند. بعد، عملیات ناهمگام واقعی را شروع کنید. در نهایت، `Task<T>` متصل به آن `TaskCompletionSource<T>` را برگردانید.

مهم است که در این الگو مطمئن شوید `TaskCompletionSource<T>` همیشه کامل می‌شود. به‌ویژه در مورد مدیریت خطاها دقیق فکر کنید و اطمینان حاصل کنید که `TaskCompletionSource<T>` به‌درستی کامل خواهد شد. در مثال آخر، استثناها صراحتاً به کال‌بک پاس داده می‌شوند، بنابراین نیازی به بلوک `catch` ندارید؛ اما برخی الگوهای غیرمعیار ممکن است لازم باشد استثناها را در کال‌بک‌های خودتان بگیرید و آن‌ها را روی `TaskCompletionSource<T>` قرار دهید.

برای مطالعه بیشتر:
- دستورالعمل 8.1 پوشاننده‌های TAP برای اعضای EAP (`OperationAsync`، `OperationCompleted`) را پوشش می‌دهد.
- دستورالعمل 8.2 پوشاننده‌های TAP برای اعضای APM (`BeginOperation`، `EndOperation`) را پوشش می‌دهد.

---

## 8.4. پوشاننده‌های Async برای کد موازی

### مسئله
پردازش موازی (CPU-bound) دارید که می‌خواهید آن را با `await` مصرف کنید. معمولاً این کار برای این مطلوب است که نخ رابط کاربری (UI thread) تا تکمیل پردازش موازی مسدود نشود.

### راه‌حل
نوع `Parallel` و Parallel LINQ برای انجام پردازش موازی از استخر نخ‌ها (thread pool) استفاده می‌کنند. آن‌ها نخ فراخوان (calling thread) را نیز به‌عنوان یکی از نخ‌های پردازش موازی در نظر می‌گیرند، بنابراین اگر یک متد موازی را از نخ UI صدا بزنید، رابط کاربری تا زمانی که پردازش کامل شود بی‌پاسخ می‌ماند.

برای حفظ پاسخ‌گویی UI، پردازش موازی را داخل `Task.Run` بپیچید و نتیجه را `await` کنید:

</div>

```csharp
await Task.Run(() => Parallel.ForEach(...));
```

<div dir="rtl">

نکته کلیدی این دستورالعمل این است که کد موازی، نخ فراخوان را نیز در مجموعه نخ‌هایی که برای پردازش موازی استفاده می‌کند، لحاظ می‌نماید. این موضوع هم برای Parallel LINQ و هم برای کلاس `Parallel` صادق است.

### بحث
این یک دستورالعمل ساده است اما اغلب نادیده گرفته می‌شود. با استفاده از `Task.Run` شما تمام پردازش موازی را به استخر نخ‌ها منتقل می‌کنید. `Task.Run` یک `Task` برمی‌گرداند که نماینده آن کار موازی است و نخ UI می‌تواند به‌صورت ناهمگام منتظر تکمیل آن بماند.

این دستورالعمل فقط برای کدهای UI کاربرد دارد. در سمت سرور (مثلاً ASP.NET)، پردازش موازی به‌ندرت انجام می‌شود، چون میزبان سرور همین حالا موازی‌سازی را انجام می‌دهد. به همین دلیل، کد سمت سرور نه باید پردازش موازی انجام دهد و نه کار را به استخر نخ‌ها منتقل کند.

برای مطالعه بیشتر:

 فصل ۴ مقدمات کد موازی را پوشش می‌دهد.

 فصل ۲ مقدمات کد ناهمگام را پوشش می‌دهد.

</div>



<div dir="rtl">

## 8.5. پوشاننده‌های Async برای Observables در System.Reactive

### مسئله
یک جریان observable دارید که می‌خواهید آن را با استفاده از `await` مصرف کنید.

### راه‌حل
ابتدا مشخص کنید به کدام رویداد observable در جریان علاقه‌مندید. حالت‌های رایج:
 آخرین رویداد قبل از پایان جریان

 رویداد بعدی

 همه رویدادها

برای گرفتن آخرین رویداد در جریان، می‌توانید یا نتیجه `LastAsync` را `await` کنید یا مستقیماً خود observable را `await` کنید:

</div>

```csharp
IObservable<int> observable = ...;
int lastElement = await observable.LastAsync();
// or:
int lastElement2 = await observable;
```

<div dir="rtl">

وقتی یک observable یا `LastAsync` را `await` می‌کنید، کد تا زمانی که جریان کامل شود به‌صورت ناهمگام منتظر می‌ماند و سپس آخرین عنصر را برمی‌گرداند. در پشت صحنه، `await` در حال subscribe کردن به جریان است.

برای گرفتن رویداد بعدی در جریان، از `FirstAsync` استفاده کنید. در کد زیر، `await` به جریان subscribe می‌کند و به‌محض رسیدن اولین رویداد، کامل شده و unsubscribe می‌کند:

</div>

```csharp
IObservable<int> observable = ...;
int nextElement = await observable.FirstAsync();
```

<div dir="rtl">

برای گرفتن همه رویدادهای جریان، از `ToList` استفاده کنید:

</div>

```csharp
IObservable<int> observable = ...;
IList<int> allElements = await observable.ToList();
```

<div dir="rtl">

### بحث
کتابخانه `System.Reactive` تمام ابزارهای لازم برای مصرف جریان‌ها با استفاده از `await` را فراهم می‌کند. تنها بخش کمی tricky این است که بیندیشید آیا awaitable تا زمان کامل شدن جریان منتظر می‌ماند یا خیر. در میان مثال‌های این بخش، `LastAsync`، `ToList` و await مستقیم تا پایان جریان منتظر می‌مانند؛ `FirstAsync` فقط برای رویداد بعدی منتظر می‌ماند.

اگر این مثال‌ها نیاز شما را برآورده نمی‌کنند، تمام قدرت LINQ و عملگرهای `System.Reactive` را در اختیار دارید؛ عملگرهایی مانند `Take` و `Buffer` کمک می‌کنند بدون انتظار برای تکمیل کل جریان، به‌صورت ناهمگام برای عناصری که نیاز دارید منتظر بمانید.

نکته: برخی عملگرهایی که همراه با `await` استفاده می‌شوند — مثل `FirstAsync` و `LastAsync` — لزوماً یک `Task<T>` بازنمی‌گردانند. اگر قصد استفاده از `Task.WhenAll` یا `Task.WhenAny` را دارید، به یک `Task<T>` واقعی نیاز دارید؛ آن را با فراخوانی `ToTask` روی هر observable به دست آورید. `ToTask` یک `Task<T>` برمی‌گرداند که با آخرین مقدار در جریان کامل می‌شود.

### برای مطالعه بیشتر
 دستورالعمل 8.6 استفاده از کد ناهمگام درون یک جریان observable را پوشش می‌دهد.

 دستورالعمل 8.8 استفاده از جریان‌های observable به‌عنوان ورودی یک بلاک dataflow (که می‌تواند کار ناهمگام انجام دهد) را پوشش می‌دهد.

 دستورالعمل 6.3 پنجره‌بندی (windows) و بافر کردن برای جریان‌های observable را پوشش می‌دهد.

---

## 8.6. پوشاننده‌های Observable در System.Reactive برای کد async

### مسئله
یک عملیات ناهمگام دارید که می‌خواهید آن را با یک observable ترکیب کنید.

### راه‌حل
هر عملیات ناهمگامی را می‌توان به‌صورت یک جریان observable در نظر گرفت که یکی از دو کار را انجام می‌دهد:

 یک عنصر تولید می‌کند و سپس کامل می‌شود

 بدون تولید هیچ عنصری دچار خطا (fault) می‌شود

برای پیاده‌سازی این تبدیل، کتابخانه `System.Reactive` تبدیل ساده‌ای از `Task<T>` به `IObservable<T>` دارد. مثال زیر دانلود ناهمگام یک صفحه وب را آغاز می‌کند و آن را به‌عنوان یک دنباله observable در نظر می‌گیرد:

</div>

```csharp
IObservable<HttpResponseMessage> GetPage(HttpClient client)
{
    Task<HttpResponseMessage> task =
        client.GetAsync("http://www.example.com/");
    return task.ToObservable();
}
```

<div dir="rtl">

رویکرد `ToObservable` فرض می‌کند شما قبلاً متد async را فراخوانی کرده‌اید و یک `Task` برای تبدیل در اختیار دارید.

رویکرد دیگر فراخوانی `StartAsync` است. `StartAsync` نیز بلافاصله متد async را صدا می‌زند اما از لغو (cancellation) پشتیبانی می‌کند: اگر یک اشتراک (subscription) از بین برود (disposed)، متد async لغو می‌شود:

</div>

```csharp
IObservable<HttpResponseMessage> GetPage(HttpClient client)
{
    return Observable.StartAsync(
        token => client.GetAsync("http://www.example.com/", token));
}
```

<div dir="rtl">

هر دو `ToObservable` و `StartAsync` بلافاصله عملیات ناهمگام را بدون انتظار برای یک subscription آغاز می‌کنند؛ این observable «داغ» (hot) است. برای ایجاد یک observable «سرد» (cold) که فقط هنگام subscribe شدن عملیات را شروع کند، از `FromAsync` استفاده کنید (که مانند `StartAsync` از لغو نیز پشتیبانی می‌کند):

</div>

```csharp
IObservable<HttpResponseMessage> GetPage(HttpClient client)
{
    return Observable.FromAsync(
        token => client.GetAsync("http://www.example.com/", token));
}
```

<div dir="rtl">

`FromAsync` به‌طور قابل‌توجهی با `ToObservable` و `StartAsync` متفاوت است؛ آن‌ها یک observable برای عملیاتی که از قبل شروع شده برمی‌گردانند، اما `FromAsync` هر بار که subscribe می‌شود، یک عملیات ناهمگام جدید و مستقل را آغاز می‌کند.

در نهایت، می‌توانید از سربارگذاری‌های ویژه `SelectMany` استفاده کنید تا برای هر رویدادی که در یک جریان مبدأ می‌رسد، عملیات ناهمگام را آغاز کنید. `SelectMany` از لغو نیز پشتیبانی می‌کند.

مثال زیر یک جریان رویداد موجود از URLها را می‌گیرد و هنگام رسیدن هر URL یک درخواست را آغاز می‌کند:

</div>

```csharp
IObservable<HttpResponseMessage> GetPages(
    IObservable<string> urls, HttpClient client)
{
    return urls.SelectMany(
        (url, token) => client.GetAsync(url, token));
}
```

<div dir="rtl">

### بحث
`System.Reactive` پیش از معرفی async وجود داشت، اما این عملگرها (و سایرین) را افزود تا بتواند با کد async به‌خوبی درهم‌کنش داشته باشد. توصیه می‌شود از عملگرهای توصیف‌شده استفاده کنید، حتی اگر بتوانید همان قابلیت را با عملگرهای دیگر `System.Reactive` بسازید.

### برای مطالعه بیشتر
 دستورالعمل 8.5 مصرف جریان‌های observable با کد ناهمگام را پوشش می‌دهد.

 دستورالعمل 8.8 استفاده از بلوک‌های dataflow (که می‌توانند کد ناهمگام داشته باشند) را به‌عنوان منابع جریان‌های observable پوشش می‌دهد.

---

## 8.7. جریان‌های ناهمگام و شبکه‌های Dataflow

### مسئله
بخشی از راه‌حل شما از جریان‌های ناهمگام استفاده می‌کند و بخشی از راه‌حل از شبکه‌های dataflow استفاده می‌کند، و باید داده را بین آن‌ها عبور دهید.

### راه‌حل
مصرف یک `Channel` به‌عنوان جریان ناهمگام مستقیماً در انواع Channel تعبیه شده است؛ برای جزئیات به دستورالعمل 9.8 مراجعه کنید. مصرف یک بلوک `TPL Dataflow` به‌عنوان یک جریان ناهمگام کمی دست‌وپاگیرتر است، اما شدنی است. رویکرد پیشنهادی این است که ابتدا یک متد اکستنشن برای بلوک‌های dataflow تعریف کنیم تا API آن‌ها بیشتر شبیه Channelها شود، و سپس آن‌ها را به‌صورت یک جریان ناهمگام مصرف کنیم:

</div>

```csharp
public static class DataflowExtensions
{
    public static bool TryReceiveItem<T>(this ISourceBlock<T> block, out T? value)
    {
        if (block is IReceivableSourceBlock<T> receivableSourceBlock)
            return receivableSourceBlock.TryReceive(out value);

        try
        {
            value = block.Receive(TimeSpan.Zero);
            return true;
        }
        catch (TimeoutException)
        {
            // در حال حاضر آیتمی در دسترس نیست.
            value = default;
            return false;
        }
        catch (InvalidOperationException)
        {
            // بلوک کامل شده و آیتم دیگری وجود ندارد.
            value = default;
            return false;
        }
    }

    public static async IAsyncEnumerable<T> ReceiveAllAsync<T>(
        this ISourceBlock<T> block,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        while (await block.OutputAvailableAsync(cancellationToken).ConfigureAwait(false))
        {
            while (block.TryReceiveItem(out var value))
            {
                yield return value!;
            }
        }
    }
}
```

<div dir="rtl">

برای جزئیات مربوط به ویژگی `EnumeratorCancellation` به دستورالعمل 3.4 مراجعه کنید.

با استفاده از متد اکستنشن بالا، می‌توان هر بلوک خروجی dataflow را به‌عنوان یک جریان ناهمگام مصرف کرد:

</div>

```csharp
var multiplyBlock = new TransformBlock<int, int>(value => value * 2);

multiplyBlock.Post(5);
multiplyBlock.Post(2);
multiplyBlock.Complete();

await foreach (int item in multiplyBlock.ReceiveAllAsync())
{
    Console.WriteLine(item);
}
```

<div dir="rtl">

همچنین می‌توان از یک جریان ناهمگام به‌عنوان منبع آیتم‌ها برای یک بلوک dataflow استفاده کرد. کافی است یک حلقه، آیتم‌ها را از جریان بیرون بکشد و داخل بلوک قرار دهد. در کد زیر چند فرض وجود دارد که ممکن است همیشه مناسب نباشد: اول، فرض می‌شود می‌خواهید بلوک با کامل شدن جریان، کامل شود. دوم، حلقه روی نخ فراخوان شروع می‌شود؛ برخی سناریوها ممکن است بخواهند کل حلقه همیشه روی نخ استخر نخ‌ها اجرا شود.

</div>

```csharp
public static async Task WriteToBlockAsync<T>(
    this IAsyncEnumerable<T> enumerable,
    ITargetBlock<T> block,
    CancellationToken token = default)
{
    try
    {
        await foreach (var item in enumerable.WithCancellation(token).ConfigureAwait(false))
        {
            await block.SendAsync(item, token).ConfigureAwait(false);
        }
        block.Complete();
    }
    catch (Exception ex)
    {
        block.Fault(ex);
    }
}
```

<div dir="rtl">

### بحث
متدهای اکستنشن این بخش یک نقطه شروع هستند. به‌طور خاص، `WriteToBlockAsync` چند فرض دارد؛ پیش از استفاده، رفتار آن‌ها را با نیازهای سناریوی خود تطبیق دهید.

### برای مطالعه بیشتر
 دستورالعمل 9.8 مصرف یک Channel به‌عنوان جریان ناهمگام را پوشش می‌دهد.

 دستورالعمل 3.4 لغو جریان‌های ناهمگام را پوشش می‌دهد.

 فصل 5 دستورالعمل‌هایی برای `TPL Dataflow` ارائه می‌دهد.

 فصل 3 دستورالعمل‌هایی برای جریان‌های ناهمگام ارائه می‌دهد.

</div>



<div dir="rtl">

## 8.8. Observables در System.Reactive و شبکه‌های Dataflow

### مسئله
بخشی از راه‌حل شما از observables در System.Reactive استفاده می‌کند و بخشی دیگر از شبکه‌های dataflow، و لازم است آن‌ها با هم ارتباط برقرار کنند.  
Observables در System.Reactive و شبکه‌های dataflow هرکدام کاربردهای خود را دارند و تا حدی همپوشانی مفهومی دارند؛ این دستورالعمل نشان می‌دهد چطور به‌سادگی با هم کار می‌کنند تا بتوانید برای هر بخش از کار، بهترین ابزار را به‌کار بگیرید.

### راه‌حل
ابتدا استفاده از یک بلوک dataflow به‌عنوان ورودی یک جریان observable را در نظر بگیرید. کد زیر یک `BufferBlock<T>` می‌سازد (که پردازشی انجام نمی‌دهد) و با فراخوانی `AsObservable` از آن بلوک یک واسط observable ایجاد می‌کند:

</div>

```csharp
var buffer = new BufferBlock<int>();
IObservable<int> integers = buffer.AsObservable();

integers.Subscribe(
    data => Trace.WriteLine(data),
    ex => Trace.WriteLine(ex),
    () => Trace.WriteLine("Done"));

buffer.Post(13);
```

<div dir="rtl">

بلوک‌های بافر و جریان‌های observable می‌توانند به‌صورت عادی یا با خطا کامل شوند، و متد `AsObservable` تکمیل (یا خطای) بلوک را به تکمیل جریان observable ترجمه می‌کند. بااین‌حال، اگر بلوک با یک استثنا fault شود، آن استثنا هنگام ارسال به جریان observable درون یک `AggregateException` پیچیده می‌شود. این رفتاری شبیه به انتشار خطا در بلوک‌های لینک‌شده است.

کمی پیچیده‌تر است که یک mesh را مقصد یک جریان observable قرار دهیم. کد زیر با فراخوانی `AsObserver` یک بلوک را قادر می‌کند تا به یک جریان observable subscribe کند:

</div>

```csharp
IObservable<DateTimeOffset> ticks =
    Observable.Interval(TimeSpan.FromSeconds(1))
              .Timestamp()
              .Select(x => x.Timestamp)
              .Take(5);

var display = new ActionBlock<DateTimeOffset>(x => Trace.WriteLine(x));

ticks.Subscribe(display.AsObserver());

try
{
    display.Completion.Wait();
    Trace.WriteLine("Done.");
}
catch (Exception ex)
{
    Trace.WriteLine(ex);
}
```

<div dir="rtl">

دقیقاً مانند قبل، تکمیل جریان observable به تکمیل بلوک ترجمه می‌شود و هرگونه خطا از سوی جریان observable به fault شدن بلوک ترجمه می‌گردد.

### بحث
بلوک‌های dataflow و جریان‌های observable اشتراک مفهومی زیادی دارند. هر دو داده را عبور می‌دهند و هر دو مفهوم تکمیل و fault را درک می‌کنند. آن‌ها برای سناریوهای متفاوتی طراحی شده‌اند؛ TPL Dataflow برای ترکیبی از برنامه‌نویسی ناهمگام و موازی طراحی شده، در حالی که System.Reactive برای برنامه‌نویسی واکنشی (reactive) طراحی شده است. بااین‌حال، این همپوشانی مفهومی به‌اندازه‌ای سازگار است که آن‌ها بسیار خوب و طبیعی با هم کار می‌کنند.

### برای مطالعه بیشتر
 دستورالعمل 8.5 مصرف جریان‌های observable با کد ناهمگام را پوشش می‌دهد.

 دستورالعمل 8.6 استفاده از کد ناهمگام درون یک جریان observable را پوشش می‌دهد.

---

## 8.9. تبدیل Observables در System.Reactive به جریان‌های ناهمگام

### مسئله
بخشی از راه‌حل شما از observables در System.Reactive استفاده می‌کند و می‌خواهید آن‌ها را به‌صورت جریان‌های ناهمگام مصرف کنید.

### راه‌حل
Observables در System.Reactive مبتنی بر push هستند، و جریان‌های ناهمگام مبتنی بر pull. بنابراین از همان ابتدا یک عدم‌تطابق مفهومی وجود دارد. باید در برابر جریان observable پاسخ‌گو بمانید و اعلان‌های آن را تا زمانی که کد مصرف‌کننده درخواستشان را می‌دهد ذخیره کنید.

ساده‌ترین راه‌حل از قبل در کتابخانه `System.Linq.Async` وجود دارد:

</div>

```csharp
IObservable<long> observable =
    Observable.Interval(TimeSpan.FromSeconds(1));

// هشدار: ممکن است حافظه نامحدود مصرف کند؛ بحث را ببینید!
IAsyncEnumerable<long> enumerable =
    observable.ToAsyncEnumerable();
```

<div dir="rtl">

نکته: متد اکستنشن `ToAsyncEnumerable` در بسته NuGet به نام `System.Linq.Async` قرار دارد.

بااین‌حال، مهم است که بدانید این متد اکستنشن ساده `ToAsyncEnumerable` در پشت‌صحنه از یک صف تولیدکننده/مصرف‌کننده نامحدود استفاده می‌کند. این اساساً معادل متد اکستنشنی است که می‌توانید خودتان با استفاده از یک `Channel` به‌عنوان صف تولیدکننده/مصرف‌کننده نامحدود بنویسید:

</div>

```csharp
// هشدار: ممکن است حافظه نامحدود مصرف کند؛ بحث را ببینید!
public static async IAsyncEnumerable<T> ToAsyncEnumerable<T>(
    this IObservable<T> observable)
{
    Channel<T> buffer = Channel.CreateUnbounded<T>();

    using (observable.Subscribe(
        value => buffer.Writer.TryWrite(value),
        error => buffer.Writer.Complete(error),
        () => buffer.Writer.Complete()))
    {
        await foreach (T item in buffer.Reader.ReadAllAsync())
            yield return item;
    }
}
```

<div dir="rtl">

این‌ها راه‌حل‌های ساده‌ای هستند، اما از صف‌های نامحدود استفاده می‌کنند؛ پس فقط زمانی باید استفاده شوند که مطمئن باشید مصرف‌کننده می‌تواند (در نهایت) با رویدادهای observable همگام شود. ایرادی ندارد اگر تولیدکننده مدتی سریع‌تر از مصرف‌کننده عمل کند؛ در آن زمان، رویدادهای observable وارد بافر می‌شوند. مادامی‌که تولیدکننده نهایتاً «جبران» کند، راه‌حل‌های فوق کار خواهند کرد. اما اگر تولیدکننده همیشه سریع‌تر از مصرف‌کننده باشد، رویدادهای observable همچنان خواهند رسید، بافر بزرگ و بزرگ‌تر می‌شود و در نهایت تمام حافظه پردازه را مصرف می‌کند.

می‌توانید با استفاده از یک صف محدود از مشکل حافظه اجتناب کنید. بهای این انتخاب این است که باید تصمیم بگیرید با آیتم‌های اضافی چه کنید اگر رویدادهای observable صف را پر کردند. یکی از گزینه‌ها حذف آیتم‌های اضافه است؛ کد نمونه زیر از یک channel محدود استفاده می‌کند تا وقتی بافر پر است، قدیمی‌ترین اعلان observable را دور بیندازد:

</div>

```csharp
// هشدار: ممکن است آیتم‌ها را حذف کند؛ بحث را ببینید!
public static async IAsyncEnumerable<T> ToAsyncEnumerable<T>(
    this IObservable<T> observable, int bufferSize)
{
    var bufferOptions = new BoundedChannelOptions(bufferSize)
    {
        FullMode = BoundedChannelFullMode.DropOldest,
    };

    Channel<T> buffer = Channel.CreateBounded<T>(bufferOptions);

    using (observable.Subscribe(
        value => buffer.Writer.TryWrite(value),
        error => buffer.Writer.Complete(error),
        () => buffer.Writer.Complete()))
    {
        await foreach (T item in buffer.Reader.ReadAllAsync())
            yield return item;
    }
}
```

<div dir="rtl">

### بحث
وقتی تولیدکننده سریع‌تر از مصرف‌کننده کار می‌کند، گزینه‌های شما یا بافر کردن آیتم‌های تولیدکننده است (با فرض اینکه تولیدکننده نهایتاً عقب‌افتادگی را جبران کند)، یا محدود کردن آیتم‌های تولیدکننده. راه‌حل دوم در این بخش آیتم‌های تولیدکننده را با حذف مواردی که در بافر جا نمی‌شوند محدود می‌کند. همچنین می‌توانید آیتم‌های تولیدکننده را با استفاده از عملگرهای observable که برای این کار طراحی شده‌اند محدود کنید، مثل `Throttle` یا `Sample`؛ برای جزئیات به دستورالعمل 6.4 مراجعه کنید. بسته به نیازتان، ممکن است بهتر باشد قبل از تبدیل observable ورودی به `IAsyncEnumerable<T>` با یکی از این تکنیک‌ها، آن را Throttle یا Sample کنید.

فارغ از صف‌های محدود و نامحدود، گزینه سومی هم وجود دارد که اینجا پوشش داده نشده: استفاده از backpressure برای اطلاع دادن به جریان observable که باید تولید اعلان‌ها را تا زمانی که بافر آماده دریافت است متوقف کند. متأسفانه System.Reactive هنوز یک الگوی استاندارد برای backpressure ندارد، بنابراین در زمان نگارش این گزینه عملی نیست. Backpressure موضوعی پیچیده و ظریف است و کتابخانه‌های واکنشی در زبان‌های دیگر الگوهای متفاوتی را برای آن پیاده‌سازی کرده‌اند. باید دید آیا System.Reactive یکی از این الگوها را می‌پذیرد، الگوی خود را معرفی می‌کند، یا backpressure را حل‌نشده باقی می‌گذارد.

### برای مطالعه بیشتر
 دستورالعمل 6.4 عملگرهای System.Reactive طراحی‌شده برای محدودسازی (throttle) ورودی را پوشش می‌دهد.

 دستورالعمل 9.8 استفاده از Channel به‌عنوان صف نامحدود تولیدکننده/مصرف‌کننده را پوشش می‌دهد.
 
 دستورالعمل 9.10 استفاده از Channel به‌عنوان صف نمونه‌برداری (sampling queue) را پوشش می‌دهد که هنگام پر بودن آیتم‌ها را حذف می‌کند.

</div>
