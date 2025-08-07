
---
<div dir="rtl">

# فصل ۶: System.Reactive

## مبانی

LINQ مجموعه‌ای از قابلیت‌های زبانی است که به توسعه‌دهندگان امکان می‌دهد تا روی دنباله‌ها (sequences) پرس‌وجو (query) انجام دهند. دو ارائه‌دهنده (provider) رایج LINQ عبارت‌اند از:

- **LINQ به اشیاء (LINQ to Objects):** که بر پایه `IEnumerable<T>` است.
- **LINQ به موجودیت‌ها (LINQ to Entities):** که بر پایه `IQueryable<T>` است.

ارائه‌دهندگان متعددی وجود دارند و بیشتر آن‌ها ساختار کلی مشابهی دارند.

پرس‌وجوها به صورت **تنبل (lazy)** ارزیابی می‌شوند و دنباله‌ها مقدارها را به هنگام نیاز تولید می‌کنند. به طور مفهومی، این مدل یک مدل «کشیدن» (pull model) است؛ یعنی هنگام ارزیابی، آیتم‌های داده از پرس‌وجو به صورت تکی کشیده می‌شوند.

`System.Reactive` (که به اختصار Rx گفته می‌شود)، **رویدادها (events)** را به عنوان دنباله‌ای از داده‌ها که در طول زمان می‌رسند، در نظر می‌گیرد. بنابراین می‌توان Rx را به عنوان **LINQ به رویدادها (LINQ to Events)** مبتنی بر `IObservable<T>` تصور کرد.

تفاوت اصلی بین observableها (مشاهده‌پذیرها) و سایر ارائه‌دهندگان LINQ این است که Rx یک مدل «فشار دادن» (push model) است؛ یعنی در این مدل، پرس‌وجو تعیین می‌کند که برنامه چگونه در هنگام رسیدن رویدادها واکنش نشان دهد.

Rx بر پایه LINQ ساخته شده و برخی اپراتورهای قدرتمند جدید را به صورت متدهای توسعه (extension methods) اضافه می‌کند.

این فصل به بررسی برخی عملیات رایج Rx می‌پردازد. باید در نظر داشت که همه اپراتورهای LINQ نیز در اینجا قابل استفاده هستند؛ بنابراین عملیات ساده‌ای مانند **فیلتر کردن (Where)** و **تصویرسازی (Select)** از نظر مفهومی مشابه دیگر ارائه‌دهندگان LINQ عمل می‌کنند.

ما در اینجا به این عملیات رایج LINQ نخواهیم پرداخت؛ بلکه بر قابلیت‌های جدیدی که Rx بر LINQ افزوده است، به‌ویژه آن‌هایی که با «زمان (time)» مرتبط‌اند، تمرکز خواهیم کرد.

برای استفاده از System.Reactive، بسته NuGet آن را در اپلیکیشن خود نصب کنید.

---

## ۶.۱ تبدیل رویدادهای دات‌نت (Converting .NET Events)

### مشکل


---

<div dir="rtl">

### مشکل

شما رویدادی (event) دارید که نیاز دارید آن را به عنوان یک جریان ورودی (input stream) در System.Reactive در نظر بگیرید، به‌گونه‌ای که هر بار که رویداد فعال می‌شود، داده‌ای از طریق OnNext تولید شود.

### راه‌حل

کلاس `Observable` چندین تبدیل‌کننده رویداد تعریف می‌کند. بیشتر رویدادهای فریم‌ورک دات‌نت با `FromEventPattern` سازگار هستند، اما اگر رویدادهایی دارید که الگوی معمول را دنبال نمی‌کنند، می‌توانید از `FromEvent` استفاده کنید.

**FromEventPattern** بهترین کارایی را زمانی دارد که نوع نماینده‌ی (delegate) رویداد، `EventHandler<T>` باشد. بسیاری از انواع جدیدتر فریم‌ورک از این نوع نماینده استفاده می‌کنند. به عنوان مثال، نوع `Progress<T>` رویداد `ProgressChanged` را دارد که از نوع `EventHandler<T>` است و بنابراین به آسانی می‌توان آن را با FromEventPattern پوشش داد:

<div dir="ltr" align="left">


```csharp
var progress = new Progress<int>();
IObservable<EventPattern<int>> progressReports =
    Observable.FromEventPattern<int>(
        handler => progress.ProgressChanged += handler,
        handler => progress.ProgressChanged -= handler);
progressReports.Subscribe(data => Trace.WriteLine("OnNext: " + data.EventArgs));
```
</div>

توجه کنید که در اینجا، `data.EventArgs` قوی‌نوع (strongly typed) به صورت `int` است. آرگومان نوعی که به `FromEventPattern` می‌دهیم (در مثال بالا `int`) همان نوع `T` در `EventHandler<T>` است. دو آرگومان lambda به `FromEventPattern` اجازه می‌دهد که بتواند به رویداد مشترک (subscribe) و لغو اشتراک (unsubscribe) شود.

فریم‌ورک‌های رابط کاربری جدید از `EventHandler<T>` استفاده می‌کنند و به‌راحتی می‌توان آن‌ها را با `FromEventPattern` به کار برد. اما انواع قدیمی‌تر معمولاً نوع نماینده (delegate type) منحصربه‌فردی برای هر رویداد تعریف می‌کنند. با این حال، آن‌ها هم می‌توانند با `FromEventPattern` به کار روند، ولی این کار نیاز به کمی تنظیم بیشتر دارد. برای مثال، نوع `System.Timers.Timer` رویداد `Elapsed` دارد که از نوع `ElapsedEventHandler` است. می‌توان رویدادهای قدیمی‌تر را به شکل زیر با `FromEventPattern` پوشش داد:

<div dir="ltr" align="left">


```csharp
var timer = new System.Timers.Timer(interval: 1000) { Enabled = true };
IObservable<EventPattern<ElapsedEventArgs>> ticks =
    Observable.FromEventPattern<ElapsedEventHandler, ElapsedEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => timer.Elapsed += handler,
        handler => timer.Elapsed -= handler);
ticks.Subscribe(data => Trace.WriteLine("OnNext: " + data.EventArgs.SignalTime));

```

</div>

در این مثال باز هم `data.EventArgs` به صورت قوی‌نوع باقی می‌ماند. آرگومان‌های نوعی به `FromEventPattern` هم‌اکنون نوع نماینده منحصر به‌فرد و نوع مشتق‌شده از `EventArgs` هستند. آرگومان اول lambda به `FromEventPattern` تبدیل‌کننده‌ای از `EventHandler<ElapsedEventArgs>` به `ElapsedEventHandler` است؛ این تبدیل‌کننده نباید بیش از عبور دادن خود رویداد کار انجام دهد.

این نحو کمی پیچیده است. گزینه‌ی دیگر استفاده از بازتاب (Reflection) است:

<div dir="ltr" align="left">


```csharp
var timer = new System.Timers.Timer(interval: 1000) { Enabled = true };
IObservable<EventPattern<object>> ticks =
    Observable.FromEventPattern(timer, nameof(Timer.Elapsed));
ticks.Subscribe(data => Trace.WriteLine("OnNext: " + ((ElapsedEventArgs)data.EventArgs).SignalTime));
```

با این روش، فراخوانی `FromEventPattern` بسیار ساده‌تر می‌شود. توجه داشته باشید که یک نکته منفی این روش این است که مصرف‌کننده داده به صورت قوی‌نوع دریافت نمی‌کند؛ چون `data.EventArgs` از نوع `object` است و شما باید خودتان آن را به `ElapsedEventArgs` تبدیل (cast) کنید.

</div>
---

### بحث

رویدادها منبع رایجی برای داده‌ها در جریان‌های System.Reactive هستند. این روش به شما امکان پوشش دادن رویدادهایی که با الگوی استاندارد رویداد (event pattern) مطابقت دارند می‌دهد (که اولین آرگومان فرستنده و دومین آرگومان نوع آرگومان رویداد است). اگر رویدادهای نامتعارف دارید، می‌توانید همچنان از بارگیری‌های `Observable.FromEvent` استفاده کنید تا آن‌ها را به Observable تبدیل کنید.

وقتی رویدادها به Observable تبدیل می‌شوند، `OnNext` هر بار که رویداد فعال می‌شود فراخوانی می‌شود. وقتی با `AsyncCompletedEventArgs` کار می‌کنید این می‌تواند باعث رفتار غیرمنتظره‌ای شود، زیرا هر استثنایی به عنوان داده OnNext منتقل می‌شود نه به عنوان خطا OnError. برای مثال، به این پوشش‌دهنده (wrapper) برای `WebClient.DownloadStringCompleted` توجه کنید:

<div dir="ltr" align="left">


```csharp
var client = new WebClient();
IObservable<EventPattern<object>> downloadedStrings =
    Observable.FromEventPattern(client, nameof(WebClient.DownloadStringCompleted));
downloadedStrings.Subscribe(
    data =>
    {
        var eventArgs = (DownloadStringCompletedEventArgs)data.EventArgs;
        if (eventArgs.Error != null)
            Trace.WriteLine("OnNext: (Error) " + eventArgs.Error);
        else
            Trace.WriteLine("OnNext: " + eventArgs.Result);
    },
    ex => Trace.WriteLine("OnError: " + ex.ToString()),
    () => Trace.WriteLine("OnCompleted"));
client.DownloadStringAsync(new Uri("http://invalid.example.com/"));
```

</div>

زمانی که `WebClient.DownloadStringAsync` با خطا خاتمه می‌یابد، رویداد با استثنایی در `AsyncCompletedEventArgs.Error` فراخوانی می‌شود. متأسفانه، System.Reactive این رویداد را به عنوان داده می‌بیند، بنابراین اگر کد بالا را اجرا کنید، به جای اینکه `OnError:` ببینید، `OnNext: (Error)` چاپ می‌شود.

برخی اشتراک‌ها (subscribe) و لغو اشتراک‌ها (unsubscribe) در رویدادها باید از یک زمینه (context) خاص انجام شود. برای مثال، رویدادهای بسیاری از کنترل‌های رابط کاربری باید از رشته (thread) رابط کاربری مشترک شوند. System.Reactive اپراتوری دارد که کنترل زمینه اشتراک و لغو اشتراک را بر عهده می‌گیرد: `SubscribeOn`. اپراتور `SubscribeOn` در بیشتر مواقع ضروری نیست، زیرا معمولا اشتراک‌گذاری در یک رابط کاربری از رشته رابط کاربری انجام می‌شود.

> **نکته:**  
> `SubscribeOn` زمینه کدی را کنترل می‌کند که به افزودن و حذف هندلرهای رویداد می‌پردازد. این را با `ObserveOn` اشتباه نگیرید، چون `ObserveOn` زمینه اعلان‌های observable (نماینده‌هایی که به Subscribe داده می‌شوند) را کنترل می‌کند.

---

## ۶.۲ ارسال اعلان‌ها به یک زمینه (Context)

### مشکل

System.Reactive سعی می‌کند به صورت مستقل از رشته‌ها (thread agnostic) عمل کند. بنابراین اعلان‌ها (مثل `OnNext`) را در هر رشته‌ای که فعال است، اجرا می‌کند. هر اعلان OnNext به صورت متوالی اجرا می‌شود، اما لزوماً در همان رشته نیست.

معمولاً می‌خواهید این اعلان‌ها در یک زمینه خاص اجرا شوند. برای مثال، المان‌های رابط کاربری (UI) باید فقط از رشته رابط کاربری که مالک آن است اصلاح شوند، پس اگر در پاسخ به اعلان‌هایی که در یک رشته ThreadPool می‌آیند، می‌خواهید UI را به‌روزرسانی کنید، باید به رشته UI منتقل شوید.

### راه‌حل

System.Reactive اپراتور `ObserveOn` را برای انتقال اعلان‌ها به زمان‌بندی‌کننده‌ای (scheduler) دیگر فراهم می‌کند.

مثال زیر را در نظر بگیرید که با استفاده از اپراتور Interval، هر ثانیه یک اعلان OnNext تولید می‌کند:

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    Trace.WriteLine($"رشته UI برابر است با {Environment.CurrentManagedThreadId}");
    Observable.Interval(TimeSpan.FromSeconds(1))
        .Subscribe(x => Trace.WriteLine($"Interval {x} در رشته {Environment.CurrentManagedThreadId}"));
}
```

</div>

در سیستم من، خروجی به شکل زیر است:

```sql
رشته UI برابر است با 9
Interval 0 در رشته 10
Interval 1 در رشته 10
Interval 2 در رشته 11
Interval 3 در رشته 11
Interval 4 در رشته 10
Interval 5 در رشته 11
Interval 6 در رشته 11
```

چون اپراتور `Interval` بر پایه تایمر است (بدون رشته خاص)، اعلان‌ها روی یک رشته ThreadPool اجرا می‌شوند، نه رشته UI. اگر نیاز دارید یک المان UI را به‌روزرسانی کنید، می‌توانید اعلان‌ها را با `ObserveOn` به زمینه زمان‌بندی‌کننده‌ای که نمایانگر رشته UI است منتقل کنید:

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    SynchronizationContext uiContext = SynchronizationContext.Current;
    Trace.WriteLine($"رشته UI برابر است با {Environment.CurrentManagedThreadId}");
    Observable.Interval(TimeSpan.FromSeconds(1))
        .ObserveOn(uiContext)
        .Subscribe(x => Trace.WriteLine($"Interval {x} در رشته {Environment.CurrentManagedThreadId}"));
}
```
</div>

استفاده معمول دیگر از `ObserveOn` زمانی است که بخواهید در مواقع لازم از رشته UI خارج شوید. فرض کنید در وضعیتی هستید که باید هر بار حرکت موس مقداری محاسبه سنگین پردازشی (CPU-intensive) انجام دهد. به صورت پیش‌فرض، همه حرکت‌های موس در رشته UI فعال می‌شوند، بنابراین می‌توانید با `ObserveOn` آن اعلان‌ها را به یک رشته ThreadPool منتقل کنید، محاسبات را انجام دهید و سپس نتیجه را دوباره به رشته UI منتقل کنید:

<div dir="ltr" align="left">


```csharp
SynchronizationContext uiContext = SynchronizationContext.Current;
Trace.WriteLine($"رشته UI برابر است با {Environment.CurrentManagedThreadId}");
Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Select(evt => evt.EventArgs.GetPosition(this))
    .ObserveOn(Scheduler.Default)
    .Select(position =>
    {
        // محاسبه پیچیده
        Thread.Sleep(100);
        var result = position.X + position.Y;
        var thread = Environment.CurrentManagedThreadId;
        Trace.WriteLine($"نتیجه محاسبه شده {result} در رشته {thread}");
        return result;
    })
    .ObserveOn(uiContext)
    .Subscribe(x => Trace.WriteLine($"نتیجه {x} در رشته {Environment.CurrentManagedThreadId}"));
```
</div>

اگر این نمونه را اجرا کنید، خواهید دید که محاسبات روی رشته ThreadPool انجام می‌شود و نتایج روی رشته UI چاپ می‌شود. البته توجه کنید که محاسبات و نتایج کمی عقب‌تر از ورودی خواهند بود؛ زیرا به دلیل اینکه موقعیت موس بیشتر از هر ۱۰۰ میلی‌ثانیه به‌روزرسانی می‌شود، این محاسبات صف‌بندی (queue) می‌شوند. System.Reactive چندین تکنیک برای مدیریت این وضعیت دارد؛ یکی از متداول‌ترین آن‌ها که در دستور العمل ۶.۴ بررسی شده، **کاهش نرخ (throttling)** ورودی‌ها است.

---

### بحث

اپراتور `ObserveOn` در واقع اعلان‌ها را به یک زمان‌بندی‌کننده System.Reactive منتقل می‌کند. این دستورالعمل، زمان‌بندی‌کننده پیش‌فرض (رشته ThreadPool) و یک روش برای ایجاد زمان‌بندی‌کننده رابط کاربری (UI scheduler) را پوشش داد. رایج‌ترین کاربردهای `ObserveOn` انتقال به داخل یا خارج از رشته UI است، اما زمان‌بندی‌کننده‌ها در سناریوهای دیگر نیز کاربرد دارند. یکی از سناریوهای پیشرفته‌تر، شبیه‌سازی گذر زمان در آزمایش واحد (unit testing) است که در دستور العمل ۷.۶ به آن پرداخته شده است.

> **نکته:**  
> `ObserveOn` زمینه (کانتکست) اعلان‌های observable را کنترل می‌کند. این را با `SubscribeOn` اشتباه نگیرید، چرا که `SubscribeOn` زمینه کدی را کنترل می‌کند که هندلرهای رویداد اضافه و حذف می‌کند.

---

### مطالعه بیشتر

<div dir="rtl" align="right">


- دستور العمل ۶.۱ شرح می‌دهد چگونه دنباله‌ها (sequences) را از رویدادها ایجاد کنید و همچنین درباره `SubscribeOn`.
- دستور العمل ۶.۴ درباره کاهش نرخ (throttling) جریان رویدادها است.
- دستور العمل ۷.۶ درباره زمان‌بندی‌کننده مخصوصی است که برای تست کد System.Reactive استفاده می‌شود.

</div>

</div>

---


</div>





---

<div dir="rtl">

## ۶.۳ گروه‌بندی داده‌های رویداد با استفاده از Windows و Buffers

### مشکل

شما یک دنباله از رویدادها دارید و می‌خواهید رویدادهای ورودی را به محض رسیدن، گروه‌بندی کنید. به عنوان مثال، نیاز دارید به جفت‌های ورودی واکنش نشان دهید یا به تمام ورودی‌ها در یک بازه زمانی دو ثانیه پاسخ دهید.

### راه‌حل

System.Reactive دو اپراتور را برای گروه‌بندی دنباله‌های ورودی فراهم می‌کند: **Buffer** و **Window**.  
اپراتور `Buffer` رویدادهای ورودی را نگه می‌دارد تا زمانی که گروه کامل شود، سپس همه را به صورت یک مجموعه (collection) از رویدادها به صورت یکجا ارسال می‌کند. اما `Window` به صورت منطقی رویدادهای ورودی را گروه‌بندی می‌کند ولی همانطور که رویدادها می‌رسند آن‌ها را عبور می‌دهد.


<div dir="rtl" align="right">


- نوع بازگشتی `Buffer` برابر `IObservable<IList<T>>` (جریان رویداد از مجموعه‌ها) است؛
- نوع بازگشتی `Window` برابر `IObservable<IObservable<T>>` (جریان رویداد از جریان‌های رویداد) می‌باشد.

</div>

#### مثال (Buffer دو تایی):

<div dir="ltr" align="left">


```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
    .Buffer(2)
    .Subscribe(x => Trace.WriteLine($"{DateTime.Now.Second}: دریافت شد {x[0]} و {x[1]}"));
```
</div>

در سیستم من، این کد هر دو ثانیه یک جفت خروجی تولید می‌کند:

```makefile
13: دریافت شد 0 و 1
15: دریافت شد 2 و 3
17: دریافت شد 4 و 5
19: دریافت شد 6 و 7
21: دریافت شد 8 و 9
```

#### مثال (Window دو تایی):

<div dir="ltr" align="left">


```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
    .Window(2)
    .Subscribe(group =>
    {
        Trace.WriteLine($"{DateTime.Now.Second}: شروع گروه جدید");
        group.Subscribe(
            x => Trace.WriteLine($"{DateTime.Now.Second}: دریافت مقدار {x}"),
            () => Trace.WriteLine($"{DateTime.Now.Second}: پایان گروه"));
    });
```
</div>

در سیستم من، خروجی این مثال `Window` به شکل زیر است:

```makefile
17: شروع گروه جدید
18: دریافت مقدار 0
19: دریافت مقدار 1
19: پایان گروه
19: شروع گروه جدید
20: دریافت مقدار 2
21: دریافت مقدار 3
21: پایان گروه
21: شروع گروه جدید
22: دریافت مقدار 4
23: دریافت مقدار 5
23: پایان گروه
23: شروع گروه جدید
```

<div dir="rtl" align="right">


این مثال‌ها تفاوت بین **Buffer** و **Window** را نشان می‌دهد:
 **Buffer** تا زمانی که همه رویدادهای گروه دریافت شوند صبر می‌کند و سپس یک مجموعه واحد منتشر می‌کند.
 **Window** رویدادها را مشابه گروه‌بندی انجام می‌دهد، اما رویدادها را به محض رسیدن منتشر می‌کند؛ یعنی بلافاصله یک observable منتشر می‌کند که رویدادهای آن دسته را منتشر می‌کند.

</div>

---

#### استفاده با بازه‌های زمانی

هر دو `Buffer` و `Window` با بازه‌های زمانی (time spans) نیز کار می‌کنند. نمونه زیر، تمامی رویدادهای حرکت موس را در بازه‌های زمانی یک ثانیه‌ای جمع‌آوری می‌کند:

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Buffer(TimeSpan.FromSeconds(1))
    .Subscribe(x => Trace.WriteLine($"{DateTime.Now.Second}: مشاهده {x.Count} مورد."));
}
```
</div>

بسته به نحوه حرکت موس، خروجی مانند زیر خواهید دید:

```makefile
49: مشاهده 93 مورد.
50: مشاهده 98 مورد.
51: مشاهده 39 مورد.
52: مشاهده 0 مورد.
53: مشاهده 4 مورد.
54: مشاهده 0 مورد.
55: مشاهده 58 مورد.
```

---

### بحث

Buffer و Window از ابزارهایی هستند که به شما کمک می‌کنند ورودی‌ها را کنترل و شکل دهید تا به فرم دلخواه برسند. یکی دیگر از تکنیک‌های مفید، **کاهش نرخ (throttling)** است که در دستورالعمل ۶.۴ به آن خواهید پرداخت.

هر دو `Buffer` و `Window` بارگذاری‌های (overload) دیگری دارند که می‌توانند در سناریوهای پیشرفته‌تر استفاده شوند.  
بارگذاری‌هایی که پارامترهای skip و timeShift دارند، امکان ایجاد گروه‌هایی که با گروه‌های دیگر هم‌پوشانی دارند یا فاصله‌ای بین گروه‌ها وجود دارد را می‌دهند. همچنین، بارگذاری‌هایی با استفاده از نماینده‌ها (delegates) وجود دارد که به شما امکان می‌دهد مرزهای گروه‌بندی را به صورت پویا تعریف کنید.

---

### مطالعه بیشتر

<div dir="rtl" align="right">


- دستورالعمل ۶.۱ نحوه ایجاد دنباله‌ها از رویدادها را پوشش می‌دهد.
- دستورالعمل ۶.۴ به کاهش نرخ جریان رویدادها می‌پردازد.

</div>

</div>

---




---

<div dir="rtl">

## ۶.۴ مهار جریان‌های رویدادی با تکنیک‌های Throttling و Sampling

### مشکل
یکی از مشکلات رایج در کدهای Reactive زمانی است که رویدادها خیلی سریع و پشت‌سرهم می‌آیند، که می‌تواند پردازش برنامه شما را تحت فشار قرار دهد.

### راه‌حل
System.Reactive اپراتورهایی برای کنترل سیل داده‌های رویدادی ارائه می‌دهد.  
**اپراتورهای Throttle و Sample** دو راه مختلف برای مهار رویدادهای پرشتاب هستند.

#### Throttle
اپراتور Throttle یک پنجره زمانی لغزان تعیین می‌کند و با رسیدن هر رویداد، این پنجره مجدد تنظیم می‌شود. فقط وقتی پنجره زمانی منقضی شود، آخرین مقدار رویدادی را که در آن بازه دریافت شده، ارسال می‌کند.

مثال (گزارش آخرین موقعیت موس پس از یک ثانیه سکون موس):

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Select(x => x.EventArgs.GetPosition(this))
    .Throttle(TimeSpan.FromSeconds(1))
    .Subscribe(x => Trace.WriteLine($"{DateTime.Now.Second}: مشاهده {x.X + x.Y}"));
}

```
</div>

خروجی نمونه (وابسته به حرکت موس):

```makefile
47: مشاهده 139
49: مشاهده 137
51: مشاهده 424
56: مشاهده 226
```

**کاربرد رایج Throttle:** جستجوی خودکار (autocomplete)، زمانی که نمی‌خواهید جستجو تا وقتی کاربر تایپ را متوقف نکرده اجرا شود.

---

#### Sample
اپراتور Sample یک بازه زمانی مشخص دارد و در پایان هر بازه، آخرین مقدار موجود را منتشر می‌کند (اگر وجود داشته باشد). اگر داده‌ای در آن بازه دریافت نشده باشد، مقداری هم ارسال نمی‌کند.

مثال (ثبت موقعیت موس هر یک ثانیه):

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Select(x => x.EventArgs.GetPosition(this))
    .Sample(TimeSpan.FromSeconds(1))
    .Subscribe(x => Trace.WriteLine($"{DateTime.Now.Second}: مشاهده {x.X + x.Y}"));
}
```
</div>

خروجی نمونه (ابتدا موس ثابت، سپس حرکت):

```makefile
12: مشاهده 311
17: مشاهده 254
18: مشاهده 269
19: مشاهده 342
20: مشاهده 224
21: مشاهده 277
```

---

### بحث
تکنیک‌های Throttling و Sampling برای مدیریت جریان‌های پرسرعت ضروری هستند. می‌توان اپراتورهای Throttle و Sample را نوعی Where زمانی دانست.  
همچنین Where (فیلتر) نیز برای مهار داده‌های نامطلوب کاربرد دارد.

---

### مطالعه بیشتر

<div dir="rtl" align="right">


- دستورالعمل ۶.۱ نحوه ایجاد دنباله‌ها از رویدادها را پوشش می‌دهد.
- دستورالعمل ۶.۲ نحوه تغییر زمینه اجرای رویدادها را شرح می‌دهد.

</div>

---

## ۶.۵ تایم‌اوت‌ها (Timeouts)

### مشکل
انتظار دارید یک رویداد در مدت معین برسد و در صورت عدم این اتفاق، برنامه باید به موقع پاسخ بدهد (مثلاً یک درخواست وب یا عملیات ناهمزمان).

### راه‌حل
اپراتور `Timeout` یک پنجره زمانی لغزان روی جریان ورودی می‌گذارد؛ با هر رویداد تازه این پنجره ریست می‌شود. اگر هیچ رویدادی نیاید و تایم‌اوت بگذرد، با ارسال OnError و یک Exception حالت استثنا گزارش می‌شود.

#### مثال ۱: تایم‌اوت روی یک درخواست وب

<div dir="ltr" align="left">

```csharp
void GetWithTimeout(HttpClient client)
{
    client.GetStringAsync("http://www.example.com/").ToObservable()
        .Timeout(TimeSpan.FromSeconds(1))
        .Subscribe(
            x => Trace.WriteLine($"{DateTime.Now.Second}: دریافت طول داده {x.Length}"),
            ex => Trace.WriteLine(ex));
}
```
**کاربرد رایج:** عملیات‌های ناهمزمان مانند درخواست‌های وب. اما می‌توان روی هر جریان رویدادی هم استفاده کرد.

</div>

---

#### مثال ۲: تایم‌اوت روی جریان رویداد (حرکت موس)


<div dir="ltr" align="left">

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Select(x => x.EventArgs.GetPosition(this))
    .Timeout(TimeSpan.FromSeconds(1))
    .Subscribe(
        x => Trace.WriteLine($"{DateTime.Now.Second}: مشاهده {x.X + x.Y}"),
        ex => Trace.WriteLine(ex));
}
```

</div>

نتیجه پس از توقف موس بیش از یک ثانیه:

```makefile
16: مشاهده 180
16: مشاهده 178
16: مشاهده 177
16: مشاهده 176
System.TimeoutException: عملیات به پایان مهلت خود رسیده است.
```
**نکته:** بعد از رخداد TimeoutException، جریان پایان یافته و دیگر رویدادی دریافت نمی‌شود.

---

#### مثال ۳: جایگزینی جریان دوم بعد از تایم‌اوت

اگر نمی‌خواهید پایان جریان، برنامه را متوقف کند، می‌توانید رشته (Observable) دیگری را بعد از پایان تایم‌اوت جایگزین کنید:

<div dir="ltr" align="left">


```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    IObservable<Point> clicks =
        Observable.FromEventPattern<MouseButtonEventHandler, MouseButtonEventArgs>(
            handler => (s, a) => handler(s, a),
            handler => MouseDown += handler,
            handler => MouseDown -= handler)
        .Select(x => x.EventArgs.GetPosition(this));

    Observable.FromEventPattern<MouseEventHandler, MouseEventArgs>(
        handler => (s, a) => handler(s, a),
        handler => MouseMove += handler,
        handler => MouseMove -= handler)
    .Select(x => x.EventArgs.GetPosition(this))
    .Timeout(TimeSpan.FromSeconds(1), clicks)
    .Subscribe(
        x => Trace.WriteLine($"{DateTime.Now.Second}: مشاهده {x.X},{x.Y}"),
        ex => Trace.WriteLine(ex));
}
```

</div>

نتیجه نمونه آزمایش‌شده:

```makefile
49: مشاهده 95,39
49: مشاهده 94,39
49: مشاهده 94,38
49: مشاهده 94,37
53: مشاهده 130,141
55: مشاهده 469,4
```

---

<div dir="rtl" align="right">

### بحث
- اپراتور Timeout برای واکنش‌پذیری برنامه ضروری و بسیار کاربردی است، به خصوص در عملیات ناهمزمان.
- زمانی که Timeout رخ می‌دهد، عملیات زمینه‌ای (مثلاً درخواست وب) واقعاً متوقف/لغو نمی‌شود؛ با گذشت زمان، آن عملیات ممکن است کماکان اجرا یا پایان یابد.

---



### مطالعه بیشتر

- دستورالعمل ۶.۱: نحوه ایجاد دنباله‌ها از رویدادها
- دستورالعمل ۸.۶: پوشش کد ناهمزمان به عنوان جریان observable
- دستورالعمل ۱۰.۶: لغو اشتراک از دنباله‌ها با CancellationToken
- دستورالعمل ۱۰.۳: استفاده از CancellationToken به عنوان تایم‌اوت

</div>

</div>

---

