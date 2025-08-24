
<div dir="rtl" align="right">

# فصل ۱۳. برنامه‌ریزی

هنگامی که یک قطعه کد اجرا می‌شود، باید در یک رشته خاص در جایی اجرا شود. یک برنامه‌ریز (scheduler) یک شیء است که تصمیم می‌گیرد یک قطعه کد خاص در کجا اجرا شود. در چارچوب .NET چند نوع برنامه‌ریز مختلف وجود دارد که با تفاوت‌های جزئی توسط کدهای موازی و جریان داده استفاده می‌شوند.

توصیه می‌کنم تا جایی که ممکن است یک برنامه‌ریز را مشخص نکنید؛ مقادیر پیش‌فرض معمولاً درست هستند. برای مثال، عملگر <span dir="ltr">`await`</span> در کد ناهمگام به طور خودکار متد را در همان متن دوباره اجرا می‌کند مگر اینکه این پیش‌فرض را لغو کنید، همانطور که در نسخه ۲.۷ توضیح داده شده است. به همین ترتیب، کد واکنشی دارای متن‌های پیش‌فرض معقولی برای رویدادهای خود است که می‌توانید آن‌ها را با <span dir="ltr">`ObserveOn`</span>، همانطور که در نسخه ۶.۲ توضیح داده شده، لغو کنید.

اگر نیاز دارید کد دیگری در یک متن خاص (مثلاً متن رشته UI یا متن درخواست ASP.NET) اجرا شود، می‌توانید از دستورات برنامه‌ریزی در این فصل برای کنترل برنامه‌ریزی کد خود استفاده کنید.

## ۱۳.۱ برنامه‌ریزی کار برای استخر رشته

### مسئله

شما یک قطعه کد دارید که می‌خواهید به صراحت آن را در یک رشته استخر رشته اجرا کنید.

### راه‌حل

اکثر اوقات، شما می‌خواهید از <span dir="ltr">`Task.Run`</span> استفاده کنید که بسیار ساده است. کد زیر یک رشته استخر رشته را برای ۲ ثانیه مسدود می‌کند:

</div>

<div dir="ltr" align="left">

```csharp
Task task = Task.Run(() =>
{
    Thread.Sleep(TimeSpan.FromSeconds(2));
});
```

</div>

<div dir="rtl" align="right">

<span dir="ltr">`Task.Run`</span> همچنین مقادیر بازگشتی و لامبداهای ناهمگام را به خوبی درک می‌کند. کار بازگردانده شده توسط <span dir="ltr">`Task.Run`</span> در کد زیر پس از ۲ ثانیه با نتیجه ۱۳ کامل خواهد شد:

</div>

<div dir="ltr" align="left">

```csharp
Task<int> task = Task.Run(async () =>
{
    await Task.Delay(TimeSpan.FromSeconds(2));
    return 13;
});
```

</div>

<div dir="rtl" align="right">

<span dir="ltr">`Task.Run`</span> یک <span dir="ltr">`Task`</span> (یا <span dir="ltr">`Task<T>`</span>) را برمی‌گرداند که به طور طبیعی توسط کدهای ناهمگام یا واکنشی قابل مصرف است.

### بحث

<span dir="ltr">`Task.Run`</span> برای برنامه‌های کاربردی UI ایده‌آل است، زمانی که کار زمان‌بر را دارید که نمی‌توان آن را در رشته UI انجام داد. برای مثال، نسخه ۸.۴ از <span dir="ltr">`Task.Run`</span> برای هل دادن پردازش موازی به یک رشته استخر رشته استفاده می‌کند. با این حال، در ASP.NET از <span dir="ltr">`Task.Run`</span> استفاده نکنید مگر اینکه کاملاً مطمئن باشید که چه می‌کنید. در ASP.NET، کد مدیریت درخواست از قبل در یک رشته استخر رشته اجرا می‌شود، بنابراین هل دادن آن به رشته استخر رشته دیگر معمولاً ناکارآمد است.

<span dir="ltr">`Task.Run`</span> جایگزین مؤثری برای <span dir="ltr">`BackgroundWorker`</span>، <span dir="ltr">`Delegate.BeginInvoke`</span> و <span dir="ltr">`ThreadPool.QueueUserWorkItem`</span> است. هیچ یک از این APIهای قدیمی‌تر نباید در کدهای جدید استفاده شوند؛ کد با استفاده از <span dir="ltr">`Task.Run`</span> بسیار راحت‌تر می‌توان آن را به درستی نوشت و در طول زمان نگهداری کرد. علاوه بر این، <span dir="ltr">`Task.Run`</span> اکثر موارد استفاده را برای <span dir="ltr">`Thread`</span> پوشش می‌دهد، بنابراین اکثر استفاده‌های <span dir="ltr">`Thread`</span> می‌توانند با <span dir="ltr">`Task.Run`</span> جایگزین شوند (با استثنای نادر رشته‌های آپارتمان تک رشته‌ای).

کدهای موازی و جریان داده به طور پیش‌فرض در استخر رشته اجرا می‌شوند، بنابراین معمولاً نیازی به استفاده از <span dir="ltr">`Task.Run`</span> با کدهای اجرا شده توسط کتابخانه‌های Parallel، Parallel LINQ یا TPL Dataflow نیست.

اگر در حال انجام موازی‌سازی پویا هستید، از <span dir="ltr">`Task.Factory.StartNew`</span> به جای <span dir="ltr">`Task.Run`</span> استفاده کنید. این به این دلیل ضروری است که <span dir="ltr">`Task`</span> بازگردانده شده توسط <span dir="ltr">`Task.Run`</span> با گزینه‌های پیش‌فرض برای استفاده ناهمگام پیکربندی شده است (یعنی برای مصرف توسط کد ناهمگام یا واکنشی). همچنین از مفاهیم پیشرفته‌ای مانند وظایف والد/فرزند که در کدهای موازی پویا رایج‌تر هستند، پشتیبانی نمی‌کند.

### مراجعه کنید به

نسخه ۸.۶ مصرف کد ناهمگام (مانند کار بازگردانده شده از <span dir="ltr">`Task.Run`</span>) با کد واکنشی را پوشش می‌دهد.

نسخه ۸.۴ انتظار ناهمگام برای کد موازی را پوشش می‌دهد که به راحتی از طریق <span dir="ltr">`Task.Run`</span> انجام می‌شود.

نسخه ۴.۴ موازی‌سازی پویا را پوشش می‌دهد، سناریویی که در آن باید از <span dir="ltr">`Task.Factory.StartNew`</span> به جای <span dir="ltr">`Task.Run`</span> استفاده کنید.

## ۱۳.۲ اجرای کد با یک برنامه‌ریز وظیفه

### مسئله

شما چندین قطعه کد دارید که باید آن‌ها را به روشی خاص اجرا کنید. برای مثال، ممکن است نیاز داشته باشید همه قطعات کد در رشته UI اجرا شوند، یا فقط تعداد معینی را هر بار اجرا کنید.

این دستورالعمل در مورد چگونگی تعریف و ساخت یک برنامه‌ریز برای این قطعات کد است. اعمال واقعی آن برنامه‌ریز موضوع دو دستورالعمل بعدی است.

### راه‌حل

در .NET انواع مختلفی وجود دارند که می‌توانند برنامه‌ریزی را مدیریت کنند؛ این دستورالعمل بر <span dir="ltr">`TaskScheduler`</span> تمرکز می‌کند زیرا قابل حمل و نسبتاً آسان در استفاده است.

ساده‌ترین <span dir="ltr">`TaskScheduler`</span>، <span dir="ltr">`TaskScheduler.Default`</span> است که کارها را در استخر رشته صف می‌کند. شما به ندرت <span dir="ltr">`TaskScheduler.Default`</span> را در کد خود مشخص خواهید کرد، اما مهم است که از آن آگاه باشید، زیرا پیش‌فرض برای بسیاری از سناریوهای برنامه‌ریزی است. <span dir="ltr">`Task.Run`</span>، کدهای موازی و جریان داده همگی از <span dir="ltr">`TaskScheduler.Default`</span> استفاده می‌کنند.

می‌توانید یک متن خاص را دریافت و سپس کار را دوباره به آن برنامه‌ریزی کنید با استفاده از <span dir="ltr">`TaskScheduler.FromCurrentSynchronizationContext`</span>:

</div>

<div dir="ltr" align="left">

```csharp
TaskScheduler scheduler = TaskScheduler.FromCurrentSynchronizationContext();
```

</div>

<div dir="rtl" align="right">

این کد یک <span dir="ltr">`TaskScheduler`</span> ایجاد می‌کند تا <span dir="ltr">`SynchronizationContext`</span> فعلی را دریافت و کد را در آن متن برنامه‌ریزی کند.

<span dir="ltr">`SynchronizationContext`</span> یک نوع است که یک متن برنامه‌ریزی عمومی را نشان می‌دهد. چندین متن مختلف در چارچوب .NET وجود دارد؛ اکثر چارچوب‌های UI یک <span dir="ltr">`SynchronizationContext`</span> ارائه می‌دهند که رشته UI را نشان می‌دهد، و ASP.NET قبل از Core یک <span dir="ltr">`SynchronizationContext`</span> ارائه می‌داد که متن درخواست HTTP را نشان می‌داد.

<span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> یک نوع قدرتمند دیگر معرفی شده در .NET 4.5 است؛ این در واقع دو برنامه‌ریز مرتبط با هم است. عضو <span dir="ltr">`ConcurrentScheduler`</span> یک برنامه‌ریز است که اجازه می‌دهد چندین وظیفه همزمان اجرا شوند، تا زمانی که هیچ وظیفه‌ای در <span dir="ltr">`ExclusiveScheduler`</span> در حال اجرا نباشد. <span dir="ltr">`ExclusiveScheduler`</span> فقط کد را یک وظیفه در یک زمان اجرا می‌کند، و فقط زمانی که هیچ وظیفه‌ای در حال اجرا در <span dir="ltr">`ConcurrentScheduler`</span> نیست:

</div>

<div dir="ltr" align="left">

```csharp
var schedulerPair = new ConcurrentExclusiveSchedulerPair();
TaskScheduler concurrent = schedulerPair.ConcurrentScheduler;
TaskScheduler exclusive = schedulerPair.ExclusiveScheduler;
```

</div>

<div dir="rtl" align="right">

یک استفاده رایج از <span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> فقط استفاده از <span dir="ltr">`ExclusiveScheduler`</span> برای اطمینان از اجرای یک وظیفه در هر زمان است. کدی که در <span dir="ltr">`ExclusiveScheduler`</span> اجرا می‌شود در استخر رشته اجرا خواهد شد اما محدود به اجرای انحصاری نسبت به تمام کدهای دیگر با استفاده از همان نمونه <span dir="ltr">`ExclusiveScheduler`</span> خواهد بود.

کاربرد دیگر <span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> به عنوان یک برنامه‌ریز محدودکننده است. می‌توانید یک <span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> ایجاد کنید که همزمانی خود را محدود کند. در این سناریو، معمولاً از <span dir="ltr">`ExclusiveScheduler`</span> استفاده نمی‌شود:

</div>

<div dir="ltr" align="left">

```csharp
var schedulerPair = new ConcurrentExclusiveSchedulerPair(
    TaskScheduler.Default, maxConcurrencyLevel: 8);
TaskScheduler scheduler = schedulerPair.ConcurrentScheduler;
```

</div>

<div dir="rtl" align="right">

توجه کنید که این نوع محدودسازی فقط کد را در حال اجرا محدود می‌کند؛ این بسیار متفاوت از نوع محدودسازی منطقی پوشش داده شده در نسخه ۱۲.۵ است. به طور خاص، کد ناهمگام در حالی که در انتظار یک عملیات است، در حال اجرا محسوب نمی‌شود. <span dir="ltr">`ConcurrentScheduler`</span> کد در حال اجرا را محدود می‌کند؛ محدودسازی‌های دیگر، مانند <span dir="ltr">`SemaphoreSlim`</span>، در سطح بالاتر (یعنی یک متد async کامل) محدود می‌شوند.

### بحث

ممکن است متوجه شده باشید که مثال کد آخر <span dir="ltr">`TaskScheduler.Default`</span> را به سازنده <span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> ارسال کرد. این به این دلیل است که <span dir="ltr">`ConcurrentExclusiveSchedulerPair`</span> منطق همزمان/انحصاری خود را در اطراف یک <span dir="ltr">`TaskScheduler`</span> موجود اعمال می‌کند.

این دستورالعمل <span dir="ltr">`TaskScheduler.FromCurrentSynchronizationContext`</span> را معرفی می‌کند که برای اجرای کد در یک متن دریافت شده مفید است. همچنین امکان استفاده مستقیم از <span dir="ltr">`SynchronizationContext`</span> برای اجرای کد در آن متن وجود دارد؛ با این حال، این رویکرد را توصیه نمی‌کنم. هر زمان که ممکن است، از عملگر <span dir="ltr">`await`</span> برای ادامه در یک متن به طور ضمنی دریافت شده استفاده کنید یا از یک پوشش <span dir="ltr">`TaskScheduler`</span> استفاده کنید.

هرگز از انواع مختص پلتفرم برای اجرای کد در رشته UI استفاده نکنید. WPF، Silverlight، iOS و Android همگی نوع <span dir="ltr">`Dispatcher`</span> را ارائه می‌دهند، Universal Windows از <span dir="ltr">`CoreDispatcher`</span> استفاده می‌کند، و Windows Forms رابط <span dir="ltr">`ISynchronizeInvoke`</span> (یعنی <span dir="ltr">`Control.Invoke`</span>) را دارد. از هیچ یک از این انواع در کدهای جدید استفاده نکنید؛ فقط وانمود کنید که وجود ندارند. استفاده از آن‌ها کد شما را بیهوده به یک پلتفرم خاص وابسته می‌کند. <span dir="ltr">`SynchronizationContext`</span> یک انتزاع عمومی در اطراف این انواع است.

<span dir="ltr">`System.Reactive (Rx)`</span> یک انتزاع برنامه‌ریز عمومی‌تر معرفی می‌کند: <span dir="ltr">`IScheduler`</span>. یک برنامه‌ریز Rx قادر به پوشش هر نوع برنامه‌ریز دیگری است؛ <span dir="ltr">`TaskPoolScheduler`</span> هر <span dir="ltr">`TaskFactory`</span> (که شامل یک <span dir="ltr">`TaskScheduler`</span> است) را پوشش می‌دهد. تیم Rx همچنین یک اجرای <span dir="ltr">`IScheduler`</span> را تعریف کرده است که برای آزمایش به صورت دستی کنترل می‌شود. اگر نیاز به استفاده از یک انتزاع برنامه‌ریز دارید، <span dir="ltr">`IScheduler`</span> از Rx را توصیه می‌کنم؛ آن خوب طراحی شده، خوب تعریف شده و برای آزمایش دوستانه است. با این حال، اکثر اوقات به یک انتزاع برنامه‌ریز نیاز ندارید و کتابخانه‌های قدیمی‌تر، مانند <span dir="ltr">`Task Parallel Library (TPL)`</span> و <span dir="ltr">`TPL Dataflow`</span>، فقط نوع <span dir="ltr">`TaskScheduler`</span> را درک می‌کنند.

### مراجعه کنید به

- نسخه ۱۳.۳ اعمال <span dir="ltr">`TaskScheduler`</span> به کد موازی را پوشش می‌دهد.
- نسخه ۱۳.۴ اعمال <span dir="ltr">`TaskScheduler`</span> به کد جریان داده را پوشش می‌دهد.
- نسخه ۱۲.۵ محدودسازی منطقی سطح بالاتر را پوشش می‌دهد.
- نسخه ۶.۲ برنامه‌ریزهای <span dir="ltr">`System.Reactive`</span> برای جریان‌های رویداد را پوشش می‌دهد.
- نسخه ۷.۶ برنامه‌ریز آزمایشی <span dir="ltr">`System.Reactive`</span> را پوشش می‌دهد.

## ۱۳.۳ برنامه‌ریزی کد موازی

### مسئله

شما نیاز دارید که نحوه اجرای قطعات انفرادی کد در کد موازی را کنترل کنید.

### راه‌حل

پس از ایجاد یک نمونه <span dir="ltr">`TaskScheduler`</span> مناسب (به نسخه ۱۳.۲ مراجعه کنید)، می‌توانید آن را در گزینه‌هایی که به یک متد <span dir="ltr">`Parallel`</span> ارسال می‌کنید، بگنجانید. کد زیر یک دنباله از دنباله‌های ماتریس‌ها را دریافت می‌کند. این کد چندین حلقه موازی را شروع می‌کند و می‌خواهد همزمانی کل حلقه‌ها را صرف نظر از تعداد ماتریس‌ها در هر دنباله محدود کند:

</div>

<div dir="ltr" align="left">

```csharp
void RotateMatrices(IEnumerable<IEnumerable<Matrix>> collections, float degrees)
{
    var schedulerPair = new ConcurrentExclusiveSchedulerPair(
        TaskScheduler.Default, maxConcurrencyLevel: 8);
    TaskScheduler scheduler = schedulerPair.ConcurrentScheduler;
    ParallelOptions options = new ParallelOptions { TaskScheduler = scheduler };
    Parallel.ForEach(collections, options,
        matrices => Parallel.ForEach(matrices, options,
            matrix => matrix.Rotate(degrees)));
}
```

</div>

<div dir="rtl" align="right">

### بحث

<span dir="ltr">`Parallel.Invoke`</span> نیز یک نمونه از <span dir="ltr">`ParallelOptions`</span> را دریافت می‌کند، بنابراین می‌توانید یک <span dir="ltr">`TaskScheduler`</span> را به <span dir="ltr">`Parallel.Invoke`</span> به همان روش <span dir="ltr">`Parallel.ForEach`</span> ارسال کنید. اگر در حال انجام کد موازی پویا هستید، می‌توانید <span dir="ltr">`TaskScheduler`</span> را مستقیماً به <span dir="ltr">`TaskFactory.StartNew`</span> یا <span dir="ltr">`Task.ContinueWith`</span> ارسال کنید.

هیچ راهی برای ارسال <span dir="ltr">`TaskScheduler`</span> به کد <span dir="ltr">`Parallel LINQ (PLINQ)`</span> وجود ندارد.

### مراجعه کنید به

- نسخه ۱۳.۲ برنامه‌ریزهای وظیفه رایج و چگونگی انتخاب بین آن‌ها را پوشش می‌دهد.

## ۱۳.۴ همگام‌سازی جریان داده با استفاده از برنامه‌ریزها

### مسئله

شما نیاز دارید که نحوه اجرای قطعات انفرادی کد در کد جریان داده را کنترل کنید.

### راه‌حل

پس از ایجاد یک نمونه <span dir="ltr">`TaskScheduler`</span> مناسب (به نسخه ۱۳.۲ مراجعه کنید)، می‌توانید آن را در گزینه‌هایی که به یک بلوک جریان داده ارسال می‌کنید، بگنجانید. هنگامی که از رشته UI فراخوانی می‌شود، کد زیر یک شبکه جریان داده ایجاد می‌کند که همه مقادیر ورودی را در دو ضرب می‌کند (با استفاده از استخر رشته) و سپس مقادیر حاصل را به موارد یک جعبه لیست اضافه می‌کند (در رشته UI):

</div>

<div dir="ltr" align="left">

```csharp
var options = new ExecutionDataflowBlockOptions
{
    TaskScheduler = TaskScheduler.FromCurrentSynchronizationContext(),
};
var multiplyBlock = new TransformBlock<int, int>(item => item * 2);
var displayBlock = new ActionBlock<int>(
    result => ListBox.Items.Add(result), options);
multiplyBlock.LinkTo(displayBlock);
```

</div>

<div dir="rtl" align="right">

### بحث

مشخص کردن یک <span dir="ltr">`TaskScheduler`</span> به‌ویژه در هماهنگ کردن اقدامات بلوک‌ها در قسمت‌های مختلف شبکه جریان داده مفید است. برای مثال، می‌توانید از <span dir="ltr">`ConcurrentExclusiveSchedulerPair.ExclusiveScheduler`</span> استفاده کنید تا اطمینان حاصل کنید که بلوک‌های A و C هرگز کد را همزمان اجرا نمی‌کنند، در حالی که به بلوک B اجازه می‌دهید هر زمان که می‌خواهد اجرا شود.

به خاطر داشته باشید که همگام‌سازی توسط <span dir="ltr">`TaskScheduler`</span> فقط در حین اجرای کد اعمال می‌شود. برای مثال، اگر یک بلوک عملیاتی دارید که کد ناهمگام اجرا می‌کند و یک برنامه‌ریز انحصاری اعمال می‌کنید، کد در حالی که در انتظار است، در حال اجرا محسوب نمی‌شود.

می‌توانید یک <span dir="ltr">`TaskScheduler`</span> را برای هر نوع بلوک جریان داده مشخص کنید. حتی اگر یک بلوک کد شما را اجرا نکند (مثلاً <span dir="ltr">`BufferBlock<T>`</span>)، باز هم کارهای مدیریتی وجود دارد که باید انجام دهد و از <span dir="ltr">`TaskScheduler`</span> ارائه شده برای تمام کارهای داخلی خود استفاده خواهد کرد.

### مراجعه کنید به

- نسخه ۱۳.۲ برنامه‌ریزهای وظیفه رایج و چگونگی انتخاب بین آن‌ها را پوشش می‌دهد.

</div>
