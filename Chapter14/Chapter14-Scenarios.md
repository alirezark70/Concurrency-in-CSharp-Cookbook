
<div dir="rtl" align="right">

# فصل ۱۴. سناریوها

در این فصل، نگاهی به انواع مختلف و تکنیک‌هایی برای پرداختن به برخی سناریوهای رایج در نوشتن برنامه‌های همزمان خواهیم انداخت. این نوع سناریوها می‌توانند یک کتاب کامل را پر کنند، بنابراین فقط چند مورد را که مفیدترین یافته‌ام انتخاب کرده‌ام.

</div>

## ۱۴.۱ راه‌اندازی منابع مشترک

<div dir="rtl" align="right">

**مسئله**

شما منبعی دارید که بین بخش‌های مختلف کد مشترک است. این منبع نیاز به راه‌اندازی در اولین بار دسترسی دارد.

**راه‌حل**

چارچوب .NET نوعی را دقیقاً برای این منظور شامل می‌شود: `Lazy<T>`. شما یک نمونه از نوع `Lazy<T>` را با یک تابع کارخانه که برای راه‌اندازی نمونه استفاده می‌شود، می‌سازید. سپس نمونه از طریق ویژگی `Value` در دسترس قرار می‌گیرد. کد زیر `Lazy<T>` را نشان می‌دهد:

</div>

<div dir="ltr" align="left">

```csharp
static int _simpleValue;
static readonly Lazy<int> MySharedInteger = new Lazy<int>(() =>
_simpleValue++);
void UseSharedInteger()
{
    int sharedValue = MySharedInteger.Value;
}
```

</div>

<div dir="rtl" align="right">

صرف نظر از اینکه چند ردیف همزمان `UseSharedInteger` را فراخوانی کنند، تابع کارخانه فقط یک بار اجرا می‌شود و همه ردیف‌ها برای همان نمونه منتظر می‌مانند. پس از ایجاد، نمونه در حافظه نهان ذخیره می‌شود و دسترسی‌های بعدی به ویژگی `Value` همان نمونه را برمی‌گرداند (در مثال قبلی، `MySharedInteger.Value` همیشه ۰ خواهد بود).

رویکردی بسیار مشابه می‌تواند اگر راه‌اندازی نیاز به کار ناهمزمان داشته باشد استفاده شود؛ در این مورد، می‌توانید از `Lazy<Task<T>>` استفاده کنید:

</div>

<div dir="ltr" align="left">

```csharp
static int _simpleValue;
static readonly Lazy<Task<int>> MySharedAsyncInteger =
    new Lazy<Task<int>>(async () =>
    {
        await
        Task.Delay(TimeSpan.FromSeconds(2)).ConfigureAwait(false);
        return _simpleValue++;
    });
async Task GetSharedIntegerAsync()
{
    int sharedValue = await MySharedAsyncInteger.Value;
}
```

</div>

<div dir="rtl" align="right">

در این مثال، تابع کارخانه یک `Task<int>` را برمی‌گرداند، یعنی یک مقدار صحیح که به صورت ناهمزمان تعیین شده است. صرف نظر از اینکه چند بخش از کد همزمان `Value` را فراخوانی کنند، `Task<int>` فقط یک بار ایجاد می‌شود و به همه فراخوانی‌کنندگان برگردانده می‌شود. سپس هر فراخوانی‌کننده گزینه (ناهمزمان) انتظار تا تکمیل تسک را با ارسال تسک به `await` دارد.

کد قبلی یک الگوی پذیرفتنی است، اما برخی ملاحظات اضافی وجود دارد. برای مثال، تابع ناهمزمان ممکن است در هر ردیفی که `Value` را فراخوانی می‌کند اجرا شود و آن تابع در آن متن اجرا خواهد شد. اگر انواع مختلف ردیف وجود داشته باشند که ممکن است `Value` را فراخوانی کنند (مثلاً یک ردیف رابط کاربری و یک ردیف استخر ردیف، یا دو ردیف درخواست ASP.NET متفاوت)، ممکن است بهتر باشد تابع ناهمزمان همیشه در یک ردیف استخر اجرا شود. این کار با پیچاندن تابع کارخانه در یک فراخوانی `Task.Run` به راحتی انجام می‌شود:

</div>

<div dir="ltr" align="left">

```csharp
static int _simpleValue;
static readonly Lazy<Task<int>> MySharedAsyncInteger =
    new Lazy<Task<int>>(() => Task.Run(async () =>
    {
        await Task.Delay(TimeSpan.FromSeconds(2));
        return _simpleValue++;
    }));
async Task GetSharedIntegerAsync()
{
    int sharedValue = await MySharedAsyncInteger.Value;
}
```

</div>

<div dir="rtl" align="right">

ملاحظه دیگر این است که نمونه `Task<T>` فقط یک بار ایجاد می‌شود. اگر تابع کارخانه ناهمزمان یک استثنا پرتاب کند، `Lazy<Task<T>>` آن تسک خراب شده را در حافظه نهان ذخیره خواهد کرد. این معمولاً مطلوب نیست؛ معمولاً بهتر است تابع کارخانه را دفعه بعدی که مقدار تنبل درخواست می‌شود، مجدداً اجرا کرد تا اینکه استثنا را در حافظه نهان نگه داشت. راهی برای "بازنشانی" `Lazy<T>` وجود ندارد، اما می‌توانید یک کلاس جدید ایجاد کنید که بازسازی نمونه `Lazy<T>` را مدیریت کند:

</div>

<div dir="ltr" align="left">

```csharp
public sealed class AsyncLazy<T>
{
    private readonly object _mutex;
    private readonly Func<Task<T>> _factory;
    private Lazy<Task<T>> _instance;

    public AsyncLazy(Func<Task<T>> factory)
    {
        _mutex = new object();
        _factory = RetryOnFailure(factory);
        _instance = new Lazy<Task<T>>(_factory);
    }

    private Func<Task<T>> RetryOnFailure(Func<Task<T>> factory)
    {
        return async () =>
        {
            try
            {
                return await factory().ConfigureAwait(false);
            }
            catch
            {
                lock (_mutex)
                {
                    _instance = new Lazy<Task<T>>(_factory);
                }
                throw;
            }
        };
    }

    public Task<T> Task
    {
        get
        {
            lock (_mutex)
                return _instance.Value;
        }
    }
}

static int _simpleValue;
static readonly AsyncLazy<int> MySharedAsyncInteger =
    new AsyncLazy<int>(() => Task.Run(async () =>
    {
        await Task.Delay(TimeSpan.FromSeconds(2));
        return _simpleValue++;
    }));
async Task GetSharedIntegerAsync()
{
    int sharedValue = await MySharedAsyncInteger.Task;
}
```

</div>

<div dir="rtl" align="right">

**بحث**

آخرین نمونه کد در این دستور العمل یک الگوی کد عمومی برای راه‌اندازی تنبل ناهمزمان است و کمی ناهموار است. کتابخانه `AsyncEx` شامل یک نوع `AsyncLazy<T>` است که درست مانند یک `Lazy<Task<T>>` عمل می‌کند که تابع کارخانه خود را در استخر ردیف اجرا می‌کند و گزینه‌ای برای تلاش مجدد در صورت شکست دارد. همچنین می‌تواند مستقیماً `await` شود، بنابراین اعلام و استفاده به صورت زیر به نظر می‌رسد:

</div>

<div dir="ltr" align="left">

```csharp
static int _simpleValue;
private static readonly AsyncLazy<int> MySharedAsyncInteger =
    new AsyncLazy<int>(async () =>
    {
        await Task.Delay(TimeSpan.FromSeconds(2));
        return _simpleValue++;
    },
    AsyncLazyFlags.RetryOnFailure);

public async Task UseSharedIntegerAsync()
{
    int sharedValue = await MySharedAsyncInteger;
}
```

</div>

<div dir="rtl" align="right">

**نکته**

نوع `AsyncLazy<T>` در بسته NuGet `Nito.AsyncEx` قرار دارد.

**نگاه کنید به**

*   فصل ۱ برنامه‌نویسی async/await پایه را پوشش می‌دهد.
*   دستور العمل ۱۳.۱ برنامه‌ریزی کار در استخر ردیف را پوشش می‌دهد.

</div>



<div dir="rtl" align="right">

## ۱۴.۲ ارزیابی معوقه System.Reactive

**مسئله**

می‌خواهید هر زمان که کسی به آن مشترک می‌شود، یک منبع مشاهده‌گر جدید ایجاد کنید. برای مثال، می‌خواهید هر اشتراک نماینگر درخواست متفاوتی به یک سرویس وب باشد.

**راه‌حل**

کتابخانه `System.Reactive` عملگر `Observable.Defer` را دارد که یک تابع نماینده را هر بار که مشاهده‌گر اشتراک می‌شود اجرا می‌کند. این تابع نماینده به عنوان یک کارخانه عمل می‌کند که یک مشاهده‌گر ایجاد می‌کند. کد زیر از `Defer` برای فراخوانی یک متد ناهمزمان در هر بار اشتراک استفاده می‌کند:

</div>

<div dir="ltr" align="left">

```csharp
void SubscribeWithDefer()
{
    var invokeServerObservable = Observable.Defer(
        () => GetValueAsync().ToObservable());

    invokeServerObservable.Subscribe(_ => { });
    invokeServerObservable.Subscribe(_ => { });

    Console.ReadKey();
}

async Task<int> GetValueAsync()
{
    Console.WriteLine("Calling server...");
    await Task.Delay(TimeSpan.FromSeconds(2));
    Console.WriteLine("Returning result...");
    return 13;
}
```

</div>

<div dir="rtl" align="right">

اگر این کد را اجرا کنید، باید خروجی زیر را ببینید:

```
Calling server...
Calling server...
Returning result...
Returning result...
```

**بحث**

کد شما معمولاً بیش از یک بار به یک مشاهده‌گر مشترک نمی‌شود، اما برخی عملگرهای `System.Reactive` در پیاده‌سازی خود چنین می‌کنند. برای مثال، عملگر `Observable.While` به یک توالی منبع مادامی که شرط آن درست است، مجدداً مشترک می‌شود.

`Defer` به شما اجازه می‌دهد یک مشاهده‌گر تعریف کنید که هر زمان یک اشتراک جدید می‌آید، مجدداً ارزیابی می‌شود. این برای نیاز به تازه‌سازی یا به‌روزرسانی داده‌های آن مشاهده‌گر مفید است.

**نگاه کنید به**

*   دستور العمل ۸.۶ پیچاندن متدهای ناهمزمان در مشاهده‌گرها را پوشش می‌دهد.

</div>

<div dir="rtl" align="right">

## ۱۴.۳ اتصال داده ناهمزمان

**مسئله**

شما داده‌ها را به صورت ناهمزمان بازیابی می‌کنید و نیاز به اتصال داده‌های نتیجه دارید (مثلاً در ViewModel یک طراحی Model-View-ViewModel).

**راه‌حل**

هنگامی که یک ویژگی در اتصال داده استفاده می‌شود، باید بلافاصله و همزمان نوعی نتیجه را برگرداند. اگر مقدار واقعی نیاز به تعیین ناهمزمان دارد، می‌توانید یک نتیجه پیش‌فرض برگردانید و بعداً ویژگی را با مقدار صحیح به‌روز کنید.

به خاطر داشته باشید که عملیات ناهمزمان ممکن است شکست بخورند یا موفق شوند. از آنجا که شما یک ViewModel می‌نویسید، می‌توانید از اتصال داده برای به‌روزرسانی رابط کاربری برای شرایط خطا نیز استفاده کنید.

کتابخانه `Nito.Mvvm.Async` نوعی به نام `NotifyTask` دارد که می‌توان برای این منظور استفاده کرد:

</div>

<div dir="ltr" align="left">

```csharp
class MyViewModel
{
    public MyViewModel()
    {
        MyValue = NotifyTask.Create(CalculateMyValueAsync());
    }

    public NotifyTask<int> MyValue { get; private set; }

    private async Task<int> CalculateMyValueAsync()
    {
        await Task.Delay(TimeSpan.FromSeconds(10));
        return 13;
    }
}
```

</div>

<div dir="rtl" align="right">

امکان اتصال داده به ویژگی‌های مختلف در ویژگی `NotifyTask<T>` وجود دارد، همانطور که این مثال نشان می‌دهد:

</div>

<div dir="ltr" align="left">

```xml
<Grid>
    <Label Content="Loading..." Visibility="{Binding MyValue.IsNotCompleted, Converter={StaticResource BooleanToVisibilityConverter}}"/>
    <Label Content="{Binding MyValue.Result}" Visibility="{Binding MyValue.IsSuccessfullyCompleted, Converter={StaticResource BooleanToVisibilityConverter}}"/>
    <Label Content="An error occurred" Foreground="Red" Visibility="{Binding MyValue.IsFaulted, Converter={StaticResource BooleanToVisibilityConverter}}"/>
</Grid>
```

</div>

<div dir="rtl" align="right">

کتابخانه `MvvmCross` یک `MvxNotifyTask` دارد که بسیار شبیه به `NotifyTask<T>` است.

</div>



<div dir="rtl" align="right">

**بحث**

نوشتن یک پوشش اتصال داده خودتان به جای استفاده از کتابخانه‌ها نیز امکان‌پذیر است. کد زیر ایده اصلی را نشان می‌دهد:

</div>

<div dir="ltr" align="left">

```csharp
class BindableTask<T> : INotifyPropertyChanged
{
    private readonly Task<T> _task;

    public BindableTask(Task<T> task)
    {
        _task = task;
        var _ = WatchTaskAsync();
    }

    private async Task WatchTaskAsync()
    {
        try
        {
            await _task;
        }
        catch
        {
        }
        OnPropertyChanged("IsNotCompleted");
        OnPropertyChanged("IsSuccessfullyCompleted");
        OnPropertyChanged("IsFaulted");
        OnPropertyChanged("Result");
    }

    public bool IsNotCompleted { get { return !_task.IsCompleted; } }
    public bool IsSuccessfullyCompleted
    {
        get { return _task.Status == TaskStatus.RanToCompletion; }
    }
    public bool IsFaulted { get { return _task.IsFaulted; } }
    public T Result
    {
        get { return IsSuccessfullyCompleted ? _task.Result : default; }
    }

    public event PropertyChangedEventHandler PropertyChanged;

    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

</div>

<div dir="rtl" align="right">

توجه کنید که این یک بلوک `catch` خالی دارد عمداً: آن کد به طور خاص می‌خواهد همه استثناها را دریافت کند و آن شرایط را از طریق اتصال داده مدیریت کند. همچنین، کد به طور صریح نمی‌خواهد از `ConfigureAwait(false)` استفاده کند زیرا رویداد `PropertyChanged` باید در رشته رابط کاربری رخ دهد.

**نکته**

نوع `NotifyTask` در بسته NuGet `Nito.Mvvm.Async` قرار دارد. نوع `MvxNotifyTask` در بسته NuGet `MvvmCross` قرار دارد.

**نگاه کنید به**

*   فصل ۱ برنامه‌نویسی async/await پایه را پوشش می‌دهد.
*   دستور العمل ۲.۷ استفاده از `ConfigureAwait` را پوشش می‌دهد.

</div>

<div dir="rtl" align="right">

## ۱۴.۴ حالت ضمنی

**مسئله**

شما برخی متغیرهای حالت دارید که نیاز به دسترسی در نقاط مختلف پشته فراخوانی دارند. برای مثال، شناسه عملیات جاری که می‌خواهید برای ثبت استفاده کنید اما نمی‌خواهید آن را به عنوان پارامتر به هر متد اضافه کنید.

**راه‌حل**

بهترین راه‌حل اضافه کردن پارامترها به متدهای شما، ذخیره‌سازی داده به عنوان اعضای یک کلاس، یا استفاده از تزریق وابستگی برای ارائه داده به بخش‌های مختلف کد شماست. با این حال، در برخی موقعیت‌ها این کار می‌تواند کد را پیچیده کند.

نوع `AsyncLocal<T>` به شما اجازه می‌دهد حالت خود را در یک شیء که در یک "متن" منطقی زندگی می‌کند قرار دهید. کد زیر نحوه استفاده از `AsyncLocal<T>` برای تنظیم یک شناسه عملیات که بعداً توسط یک متد ثبت خوانده می‌شود را نشان می‌دهد:

</div>

<div dir="ltr" align="left">

```csharp
private static AsyncLocal<Guid> _operationId = new AsyncLocal<Guid>();

async Task DoLongOperationAsync()
{
    _operationId.Value = Guid.NewGuid();
    await DoSomeStepOfOperationAsync();
}

async Task DoSomeStepOfOperationAsync()
{
    await Task.Delay(100); // Some async work
    // Do some logging here.
    Trace.WriteLine("In operation: " + _operationId.Value);
}
```

</div>

<div dir="rtl" align="right">

بسیاری از اوقات، داشتن یک ساختار داده پیچیده‌تر (مانند یک پشته از مقادیر) در یک نمونه `AsyncLocal<T>` مفید است. این امکان‌پذیر است، با یک هشدار: شما باید فقط داده‌های تغییرناپذیر را در `AsyncLocal<T>` ذخیره کنید. هر زمان که نیاز به به‌روزرسانی داده دارید، باید مقدار موجود را بازنویسی کنید. اغلب پنهان کردن `AsyncLocal<T>` در یک نوع کمکی که تضمین می‌کند داده‌های ذخیره شده تغییرناپذیر و به‌روز می‌شوند، مفید است:

</div>

<div dir="ltr" align="left">

```csharp
internal sealed class AsyncLocalGuidStack
{
    private readonly AsyncLocal<ImmutableStack<Guid>> _operationIds =
        new AsyncLocal<ImmutableStack<Guid>>();

    private ImmutableStack<Guid> Current =>
        _operationIds.Value ?? ImmutableStack<Guid>.Empty;

    public IDisposable Push(Guid value)
    {
        _operationIds.Value = Current.Push(value);
        return new PopWhenDisposed(this);
    }

    private void Pop()
    {
        ImmutableStack<Guid> newValue = Current.Pop();
        if (newValue.IsEmpty)
            newValue = null;
        _operationIds.Value = newValue;
    }

    public IEnumerable<Guid> Values => Current;

    private sealed class PopWhenDisposed : IDisposable
    {
        private AsyncLocalGuidStack _stack;

        public PopWhenDisposed(AsyncLocalGuidStack stack) =>
            _stack = stack;

        public void Dispose()
        {
            _stack?.Pop();
            _stack = null;
        }
    }
}

private static AsyncLocalGuidStack _operationIds = new AsyncLocalGuidStack();

async Task DoLongOperationAsync()
{
    using (_operationIds.Push(Guid.NewGuid()))
        await DoSomeStepOfOperationAsync();
}

async Task DoSomeStepOfOperationAsync()
{
    await Task.Delay(100); // some async work
    // Do some logging here.
    Trace.WriteLine("In operation: " +
                    string.Join(":", _operationIds.Values));
}
```

</div>

<div dir="rtl" align="right">

نوع پوشش تضمین می‌کند که داده زیرین تغییرناپذیر است و مقادیر جدید به پشته اضافه می‌شوند. همچنین یک روش `IDisposable` راحت برای برداشتن مقادیر از پشته فراهم می‌کند.

**بحث**

کدهای قدیمی‌تر ممکن است از ویژگی `ThreadStatic` برای حالت متنی استفاده شده توسط کد همزمان استفاده کنند. هنگام تبدیل کدهای قدیمی به ناهمزمان، `AsyncLocal<T>` گزینه اصلی برای جایگزینی `ThreadStaticAttribute` است. `AsyncLocal<T>` هم برای کد همزمان و هم ناهمزمان کار می‌کند و باید انتخاب پیش‌فرض برای حالت ضمنی در برنامه‌های مدرن باشد.

</div>


<div dir="rtl" align="right">

**نگاه کنید به**

*   فصل ۱ برنامه‌نویسی async/await پایه را پوشش می‌دهد.
*   فصل ۹ چندین مجموعه تغییرناپذیر را پوشش می‌دهد، برای زمانی که نیاز به ذخیره‌سازی داده پیچیده به عنوان حالت ضمنی دارید.

</div>

<div dir="rtl" align="right">

## ۱۴.۵ کد همزمان و ناهمزمان یکسان

**مسئله**

شما کدی دارید که نیاز به ارائه از طریق رابط‌های API همزمان و ناهمزمان دارد، اما نمی‌خواهید منطق را تکرار کنید. اغلب در هنگام به‌روزرسانی کد به ناهمزمان با این وضعیت مواجه می‌شوید، اما مصرف‌کنندگان همزمان موجود را نمی‌توان (در حال حاضر) تغییر داد.

**راه‌حل**

اگر می‌توانید، سعی کنید کد خود را مطابق با دستورالعمل‌های طراحی مدرن مانند Ports and Adapters (معماری شش‌ضلعی) سازماندهی کنید که منطق کسب و کار را از عوارض جانبی مانند I/O جدا می‌کند. اگر بتوانید به این وضعیت برسید، نیازی به ارائه رابط‌های API همزمان و ناهمزمان برای هیچ چیز نیست؛ منطق کسب و کار شما همیشه همزمان خواهد بود و I/O همیشه ناهمزمان.

با این حال، این یک هدف بسیار والا است و در دنیای واقعی، کد قدیمی می‌تواند آشفته باشد و به ندرت زمانی وجود دارد که قبل از اتخاذ کد ناهمزمان، آن را کامل کرد. رابط‌های API موجود اغلب باید برای سازگاری معکوس حفظ شوند، حتی اگر به بدی طراحی شده باشند.

در این سناریو راه‌حل کاملی وجود ندارد. بسیاری از توسعه‌دهندگان تلاش می‌کنند کد همزمان، کد ناهمزمان را فراخوانی کند یا برعکس، اما هر دوی این رویکردها ضد الگو هستند. حقه آرگومان بولی راهکاری است که من در این موقعیت ترجیح می‌دهم. این یک روش برای نگه داشتن تمام منطق در یک متد و در عین حال ارائه رابط‌های API همزمان و ناهمزمان است.

ایده اصلی حقه آرگومان بولی این است که یک متد هسته خصوصی وجود دارد که حاوی منطق است. آن متد هسته امضای ناهمزمان دارد و یک آرگومان بولی می‌گیرد که تعیین می‌کند آیا متد هسته باید ناهمزمان باشد یا خیر. اگر آرگومان بولی مشخص کند که متد هسته باید همزمان باشد، باید یک تسک از قبل کامل شده را برگرداند. سپس می‌توانید هر دو متد API همزمان و ناهمزمان را بنویسید که به متد هسته ارجاع دهند:

</div>

<div dir="ltr" align="left">

```csharp
private async Task<int> DelayAndReturnCore(bool sync)
{
    int value = 100;
    // Do some work.
    if (sync)
        Thread.Sleep(value); // Call synchronous API.
    else
        await Task.Delay(value); // Call asynchronous API.
    return value;
}

// Asynchronous API
public Task<int> DelayAndReturnAsync() =>
    DelayAndReturnCore(sync: false);

// Synchronous API
public int DelayAndReturn() =>
    DelayAndReturnCore(sync: true).GetAwaiter().GetResult();
```

</div>

<div dir="rtl" align="right">

رابط API ناهمزمان `DelayAndReturnAsync`، `DelayAndReturnCore` را با پارامتر بولی `sync` تنظیم شده به `false` فراخوانی می‌کند؛ این یعنی `DelayAndReturnCore` ممکن است به صورت ناهمزمان رفتار کند و از `await` در API تأخیر ناهمزمان زیرین `Task.Delay` استفاده می‌کند. تسک برگردانده شده از `DelayAndReturnCore` مستقیماً به فراخوانی‌کننده `DelayAndReturnAsync` برگردانده می‌شود.

رابط API همزمان `DelayAndReturn`، `DelayAndReturnCore` را با پارامتر بولی `sync` تنظیم شده به `true` فراخوانی می‌کند؛ این یعنی `DelayAndReturnCore` باید به صورت همزمان رفتار کند و از API تأخیر همزمان زیرین `Thread.Sleep` استفاده می‌کند. تسک برگردانده شده توسط `DelayAndReturnCore` باید از قبل کامل باشد، بنابراین گرفتن نتیجه آن امن است. `DelayAndReturn` از `GetAwaiter().GetResult()` برای بازیابی نتیجه از تسک استفاده می‌کند؛ این از یک پوشش `AggregateException` که می‌تواند با استفاده از ویژگی `Task<T>.Result` رخ دهد، جلوگیری می‌کند.

**بحث**

این یک راه‌حل ایده‌آل نیست، اما می‌تواند برای برنامه‌های دنیای واقعی کمک کند.

اکنون، چند هشدار برای این راه‌حل. مخرب‌ترین مشکلات زمانی رخ می‌دهد که متد هسته پارامتر همزمان خود را به درستی رعایت نکند. اگر متد هسته هرگز یک تسک ناتمام را هنگامی که `sync` درست است برنگرداند، آنگاه رابط API همزمان به راحتی می‌تواند در بن‌بست قرار گیرد؛ تنها دلیل اینکه رابط API همزمان می‌تواند در تسک خود مسدود شود این است که می‌داند تسک از قبل کامل است. به همین ترتیب، اگر متد هسته یک رشته را هنگامی که `sync` نادرست است مسدود کند، آنگاه برنامه به اندازه کافی کارآمد نیست.

یک بهبود که می‌توان به این راه‌حل اضافه کرد، افزودن یک بررسی در رابط API همزمان است که تأیید می‌کند تسک برگردانده شده در واقع کامل است. اگر هرگز کامل نشود، یک باگ کدنویسی جدی وجود دارد.

**نگاه کنید به**

*   فصل ۱ برنامه‌نویسی async/await پایه را پوشش می‌دهد، از جمله بحثی درباره بن‌بست‌هایی که می‌توانند هنگام مسدود کردن در کد ناهمزمان به طور کلی رخ دهند.

</div>



<div dir="rtl" align="right">

## ۱۴.۶ برنامه‌نویسی راه‌آهن با شبکه‌های جریان داده

**مسئله**

شما یک شبکه جریان داده راه‌اندازی کرده‌اید، اما برخی موارد داده قابل پردازش نیستند. می‌خواهید به این خطاها به‌گونه‌ای پاسخ دهید که شبکه جریان داده شما عملیاتی بماند.

**راه‌حل**

به طور پیش‌فرض، اگر یک بلوک با برخورد به یک استثنا هنگام پردازش یک مورد داده مواجه شود، آن بلوک دچار خطا می‌شود و از پردازش سایر موارد داده جلوگیری می‌کند. ایده اصلی این راه‌حل این است که استثناها را به عنوان نوعی داده در نظر بگیرید. اگر شبکه جریان داده روی انواعی کار کند که می‌توانند هم استثنا و هم داده باشند، آنگاه شبکه می‌تواند حتی در زمان رخ دادن استثناها عملیاتی بماند و پردازش سایر موارد داده را ادامه دهد.

این گاهی اوقات "برنامه‌نویسی راه‌آهن" نامیده می‌شود زیرا موارد در شبکه را می‌توان به عنوان حرکت در یکی از دو مسیر جداگانه دید. یک مسیر عادی "داده" وجود دارد: اگر همه چیز به‌طور کامل پیش برود، مورد در مسیر "داده" باقی می‌ماند و از طریق شبکه حرکت می‌کند، تبدیل و عملیات می‌شود تا به انتهای شبکه برسد.

مسیر دوم مسیر "خطا" است؛ در هر بلوک، اگر هنگام پردازش یک مورد استثنایی رخ دهد، آن استثنا به مسیر "خطا" منتقل می‌شود و از طریق شبکه حرکت می‌کند. موارد استثنا پردازش نمی‌شوند؛ آنها فقط از یک بلوک به بلوک دیگر منتقل می‌شوند، بنابراین به انتهای شبکه نیز می‌رسند. بلوک‌های پایانی در شبکه در نهایت دنباله‌ای از موارد دریافت می‌کنند که هر کدام یا یک مورد داده یا مورد استثنا هستند؛ یک مورد داده نشان‌دهنده داده‌ای است که به‌طور کامل از شبکه با موفقیت عبور کرده، و یک مورد استثنا نشان‌دهنده یک خطای پردازش در نقطه‌ای از شبکه است.

برای راه‌اندازی این نوع "برنامه‌نویسی راه‌آهن"، ابتدا نیاز به تعریف نوعی دارید که نماینده یک مورد داده یا استثنا باشد. اگر می‌خواهید از یک نوع از پیش ساخته شده استفاده کنید، چند مورد موجود است. این نوع از نوع در جامعه برنامه‌نویسی تابعی رایج است، که معمولاً `Try` یا `Error` یا `Exceptional` نامیده می‌شود و یک مورد خاص از `Either monad` است. من نوع `Try<T>` خودم را تعریف کرده‌ام که می‌توانید به عنوان مثال استفاده کنید؛ این در بسته NuGet `Nito.Try` قرار دارد و کد منبع آن در GitHub است.

هنگامی که نوعی از `Try<T>` دارید، راه‌اندازی شبکه کمی خسته‌کننده اما وحشتناک نیست. نوع هر بلوک جریان داده باید از `T` به `Try<T>` تغییر کند و هر پردازشی در آن بلوک باید با نگاشت یک مقدار `Try<T>` به مقدار دیگر انجام شود. با نوع `Try<T>` من، این با فراخوانی `Try<T>.Map` انجام می‌شود. من پیدا کردن روش‌های کارخانه کوچک برای بلوک‌های جریان داده با جهت‌گیری راه‌آهن به جای داشتن کد اضافی درون‌خطی را مفید می‌یابم. کد زیر مثالی از یک متد کمکی است که یک `TransformBlock` را می‌سازد که روی مقادیر `Try<T>` با فراخوانی `Try<T>.Map` کار می‌کند:

</div>

<div dir="ltr" align="left">

```csharp
private static TransformBlock<Try<TInput>, Try<TOutput>>
    RailwayTransform<TInput, TOutput>(Func<TInput, TOutput>
        func)
{
    return new TransformBlock<Try<TInput>, Try<TOutput>>(t =>
        t.Map(func));
}
```

</div>

<div dir="rtl" align="right">

با کمک‌هایی مانند اینها، کد ایجاد شبکه جریان داده ساده‌تر می‌شود:

</div>

<div dir="ltr" align="left">

```csharp
var subtractBlock = RailwayTransform<int, int>(value => value - 2);
var divideBlock = RailwayTransform<int, int>(value => 60 / value);
var multiplyBlock = RailwayTransform<int, int>(value => value * 2);

var options = new DataflowLinkOptions { PropagateCompletion = true };
subtractBlock.LinkTo(divideBlock, options);
divideBlock.LinkTo(multiplyBlock, options);

// Insert data items into the first block.
subtractBlock.Post(Try.FromValue(5));
subtractBlock.Post(Try.FromValue(2));
subtractBlock.Post(Try.FromValue(4));
subtractBlock.Complete();

// Receive data/exception items from the last block.
while (await multiplyBlock.OutputAvailableAsync())
{
    Try<int> item = await multiplyBlock.ReceiveAsync();
    if (item.IsValue)
        Console.WriteLine(item.Value);
    else
        Console.WriteLine(item.Exception.Message);
}
```

</div>

<div dir="rtl" align="right">

**بحث**

برنامه‌نویسی راه‌آهن راه عالی برای جلوگیری از خطادار شدن بلوک‌های جریان داده است. از آنجا که برنامه‌نویسی راه‌آهن یک سازه برنامه‌نویسی تابعی مبتنی بر `monads` است، کمی ناهموار در ترجمه به .NET است، اما قابل استفاده است. اگر یک شبکه جریان داده دارید که نیاز به مقاوم بودن در برابر خطا دارد، پس برنامه‌نویسی راه‌آهن قطعاً ارزش آن را دارد.

**نگاه کنید به**

*   دستور العمل ۵.۲ روش معمولی که استثناها بلوک‌ها را دچار خطا می‌کنند و می‌توانند در صورت استفاده نکردن از برنامه‌نویسی راه‌آهن از طریق شبکه گسترش یابند را پوشش می‌دهد.

</div>



<div dir="rtl" align="right">

## ۱۴.۷ محدودسازی به‌روزرسانی‌های پیشرفت

**مسئله**

شما یک عملیات طولانی دارید که پیشرفت را گزارش می‌دهد و پیشرفت را در رابط کاربری نمایش می‌دهید. اما به‌روزرسانی‌های پیشرفت بسیار سریع می‌رسند و باعث می‌شوند رابط کاربری پاسخگو نباشد.

**راه‌حل**

کد زیر را در نظر بگیرید که پیشرفت را بسیار سریع گزارش می‌دهد:

</div>

<div dir="ltr" align="left">

```csharp
private string Solve(IProgress<int> progress)
{
    // Count as quickly as possible for 3 seconds.
    var endTime = DateTime.UtcNow.AddSeconds(3);
    int value = 0;
    while (DateTime.UtcNow < endTime)
    {
        value++;
        progress?.Report(value);
    }
    return value.ToString();
}
```

</div>

<div dir="rtl" align="right">

می‌توانید این کد را از یک برنامه گرافیکی با پیچاندن آن در `Task.Run` و ارسال یک `IProgress<T>` اجرا کنید. کد مثال زیر برای WPF است، اما مفاهیم مشابه برای هر پلتفرم رابط کاربری (WPF، Xamarin یا Windows Forms) صدق می‌کند:

</div>

<div dir="ltr" align="left">

```csharp
// For simplicity, this code updates a label directly.
// In a real-world MVVM application, those assignments
// would instead be updating a ViewModel property
// which is data-bound to the actual UI.
private async void StartButton_Click(object sender,
    RoutedEventArgs e)
{
    MyLabel.Content = "Starting...";
    var progress = new Progress<int>(value => MyLabel.Content = value);
    var result = await Task.Run(() => Solve(progress));
    MyLabel.Content = $"Done! Result: {result}";
}
```

</div>

<div dir="rtl" align="right">

این کد باعث می‌شود رابط کاربری برای مدت طولانی، حدود ۲۰ ثانیه در دستگاه من، پاسخگو نباشد و سپس ناگهان رابط کاربری پاسخگو می‌شود و فقط پیام "Done! Result:" را نمایش می‌دهد. گزارش‌های پیشرفت میانی هرگز دیده نشدند. آنچه رخ می‌دهد این است که کد پس‌زمینه گزارش‌های پیشرفت را به رشته رابط کاربری بسیار سریع می‌فرستد، آنقدر سریع که پس از اجرای فقط ۳ ثانیه، رشته رابط کاربری حدود ۱۷ ثانیه دیگر فقط برای پردازش همه آن گزارش‌های پیشرفت، با به‌روزرسانی مکرر آن برچسب صرف می‌کند.

در نهایت، رشته رابط کاربری برچسب را یک بار آخر با مقادیر "Done! Result:" به‌روز می‌کند و سپس سرانجام زمان برای بازنقاشی صفحه دارد و مقدار برچسب به‌روز شده را به کاربر نمایش می‌دهد.

اولین چیزی که باید درک کرد این است که نیاز به محدودسازی گزارش‌های پیشرفت داریم. این تنها راه اطمینان از زمان کافی رابط کاربری برای بازنقاشی خود بین به‌روزرسانی‌های پیشرفت است. چیز بعدی که باید درک کرد این است که می‌خواهیم محدودسازی را بر اساس زمان، نه تعداد گزارش‌ها انجام دهیم. اگرچه ممکن است وسوسه شوید گزارش‌های پیشرفت را با ارسال فقط یکی از هر صد مورد محدود کنید، این ایده‌آل نیست به دلایلی که در بخش "بحث" بررسی می‌شود.

واقعیت این است که می‌خواهیم با زمان سروکار داشته باشیم که نشان می‌دهد باید `System.Reactive` را در نظر بگیریم. و در واقع، `System.Reactive` عملگرهایی دارد که خاصاً برای محدودسازی بر اساس زمان طراحی شده‌اند. بنابراین، به نظر می‌رسد `System.Reactive` نقشی در این راه‌حل خواهد داشت.

برای شروع، می‌توانید یک پیاده‌سازی `IProgress<T>` را تعریف کنید که برای هر گزارش پیشرفت یک رویداد ایجاد می‌کند و سپس یک مشاهده‌گر ایجاد کنید که آن گزارش‌های پیشرفت را با پیچاندن آن رویداد دریافت می‌کند:

</div>

<div dir="ltr" align="left">

```csharp
public static class ObservableProgress
{
    private sealed class EventProgress<T> : IProgress<T>
    {
        void IProgress<T>.Report(T value) =>
            OnReport?.Invoke(value);

        public event Action<T> OnReport;
    }

    public static (IObservable<T>, IProgress<T>) Create<T>()
    {
        var progress = new EventProgress<T>();
        var observable = Observable.FromEvent<T>(
            handler => progress.OnReport += handler,
            handler => progress.OnReport -= handler);
        return (observable, progress);
    }
}
```

</div>

<div dir="rtl" align="right">

متد `ObservableProgress.Create<T>` یک جفت ایجاد خواهد کرد: یک `IObservable<T>` و یک `IProgress<T>`، که در آن تمام گزارش‌های پیشرفت ارسال شده به `IProgress<T>` به مشترکان `IObservable<T>` ارسال خواهند شد. اکنون یک جریان مشاهده‌گر برای گزارش‌های پیشرفت ما وجود دارد؛ مرحله بعدی محدودسازی آن است.

ما می‌خواهیم رابط کاربری را آنقدر آهسته به‌روز کنیم که پاسخگو بماند و آنقدر سریع که کاربران بتوانند به‌روزرسانی‌ها را ببینند. درک انسانی به مراتب کندتر از نمایشگرهای کامپیوتر است، بنابراین پنجره بزرگی از مقادیر ممکن وجود دارد. اگر خوانایی واقعی را ترجیح می‌دهید، محدودسازی به یک به‌روزرسانی در هر ثانیه می‌تواند کافی باشد. اگر بازخورد واقعی‌تری را ترجیح می‌دهید، من می‌یابم که یک به‌روزرسانی هر ۱۰۰ یا ۲۰۰ میلی‌ثانیه (ms) آنقدر سریع است که کاربر می‌بیند چیزی سریع اتفاق می‌افتد و درک کلی از جزئیات پیشرفت دارد، در حالی که همچنان آنقدر کند است که رابط کاربری پاسخگو بماند.

</div>


<div dir="rtl" align="right">

نکته دیگری که باید به خاطر داشت این است که گزارش‌های پیشرفت می‌توانند از رشته‌های دیگر ارسال شوند - در این مورد، آنها از یک رشته پس‌زمینه ارسال می‌شوند. محدودسازی باید تا جایی که ممکن است نزدیک به منبع انجام شود، بنابراین می‌خواهیم محدودسازی را در رشته پس‌زمینه نگه داریم.

با این حال، کدی که رابط کاربری را به‌روز می‌کند باید در رشته رابط کاربری اجرا شود. با این در نظر، می‌توانید یک متد `CreateForUi` تعریف کنید که هم محدودسازی و هم انتقال به رشته رابط کاربری را مدیریت کند:

</div>

<div dir="ltr" align="left">

```csharp
// The full implementation of CreateForUi would be similar to the original text,
// handling throttling and marshaling to the UI thread.
public static (IObservable<T>, IProgress<T>) CreateForUi<T>(TimeSpan throttleInterval)
{
    var (sourceObservable, sourceProgress) = Create<T>();
    var throttledObservable = sourceObservable
        .Sample(throttleInterval)
        .ObserveOn(SynchronizationContext.Current); // Marshal to UI thread

    // Return the throttled observable and the original progress reporter
    return (throttledObservable, sourceProgress);
}
```

</div>

<div dir="rtl" align="right">

**بحث**

این دستور العمل ترکیب سرگرم‌کننده‌ای از دستورات دیگر در این کتاب است! هیچ تکنیک جدیدی معرفی نشد؛ ما فقط مرور کردیم که چه دستوراتی را ترکیب کنیم تا به این راه‌حل برسیم.

یک راه‌حل جایگزین برای این مشکل که ممکن است در دنیای واقعی زیاد با آن مواجه شوید، "راه‌حل مدولوس" است. ایده پشت این راه‌حل این است که `Solve` خودش باید به‌روزرسانی‌های پیشرفت خود را محدود کند؛ برای مثال، اگر کد فقط می‌خواست یک به‌روزرسانی را برای هر ۱۰۰ به‌روزرسانی واقعی پردازش کند، آنگاه کد ممکن است از تکنیک مدولوس مانند `if (value % 100 == 0) progress?.Report(value);` استفاده کند.

چند مشکل با رویکرد مدولوس وجود دارد. اول اینکه هیچ مقدار مدولوس "صحیحی" وجود ندارد؛ معمولاً، توسعه‌دهنده مقادیر مختلفی را امتحان می‌کند تا در لپتاپ خودش خوب کار کند. با این حال، همین کد ممکن است هنگام اجرا در سرور بزرگ یک مشتری یا درون یک ماشین مجازی ضعیف به خوبی رفتار نکند. علاوه بر این، پلتفرم‌ها و محیط‌های مختلف بسیار متفاوت کش می‌کنند، که می‌تواند باعث شود کد بسیار سریع‌تر (یا کندتر) از حد انتظار اجرا شود. و البته، توانایی‌های سخت‌افزار "جدیدترین" کامپیوتر در طول زمان تغییر می‌کند. بنابراین مقدار مدولوس فقط به یک حدس تبدیل می‌شود؛ قرار نیست در همه جا و در طول تمام زمان صحیح باشد.

مشکل دیگر رویکرد مدولوس این است که سعی می‌کند مشکل را در بخش اشتباه کد حل کند. این مشکل صرفاً یک مسئله رابط کاربری است؛ رابط کاربری مشکل دارد و لایه رابط کاربری باید راه‌حل آن را ارائه دهد. در کد مثال این دستور العمل، `Solve` نماینده برخی از منطق پردازش کسب و کار پس‌زمینه است؛ نباید با مسائل خاص رابط کاربری مرتبط باشد. یک برنامه کنسول ممکن است بخواهد مدولوس بسیار متفاوتی نسبت به یک برنامه WPF استفاده کند.

تنها چیزی که رویکرد مدولوس در آن درست است، این است که بهتر است به‌روزرسانی‌ها را قبل از ارسال به رشته رابط کاربری محدود کرد. راه‌حل در این دستور العمل نیز این کار را انجام می‌دهد: به‌روزرسانی‌ها را بلافاصله و همزمان در رشته پس‌زمینه قبل از ارسال به رشته رابط کاربری محدود می‌کند. با تزریق پیاده‌سازی خود `IProgress<T>`، رابط کاربری قادر به انجام محدودسازی خود بدون نیاز به هیچ تغییری در متد `Solve` است.

**نگاه کنید به**

*   دستور العمل ۲.۳ استفاده از `IProgress<T>` برای گزارش پیشرفت از عملیات طولانی را پوشش می‌دهد.
*   دستور العمل ۱۳.۱ استفاده از `Task.Run` برای اجرای کد همزمان در استخر رشته را پوشش می‌دهد.
*   دستور العمل ۶.۱ استفاده از `FromEvent` برای پیچاندن رویدادهای .NET در مشاهده‌گرها را پوشش می‌دهد.
*   دستور العمل ۶.۴ استفاده از `Sample` برای محدودسازی مشاهده‌گرها بر اساس زمان را پوشش می‌دهد.
*   دستور العمل ۶.۲ استفاده از `ObserveOn` برای انتقال اطلاعیه‌های مشاهده‌گر به متن دیگری را پوشش می‌دهد.

</div>

<div dir="rtl" align="right">

# پیوست ب. شناسایی و تفسیر الگوهای ناهمزمان

مزایای کد ناهمزمان دهه‌ها قبل از اختراع .NET به خوبی درک شده بود. در روزهای اولیه .NET، چندین سبک مختلف از کد ناهمزمان توسعه یافت، در اینجا و آنجا استفاده شد و در نهایت کنار گذاشته شد. همه این‌ها ایده‌های بدی نبودند؛ بسیاری از آنها راه را برای رویکرد مدرن async/await هموار کردند. با این حال، کدهای میراثی زیادی وجود دارد که از الگوهای ناهمزمان قدیمی‌تر استفاده می‌کنند. این پیوست الگوهای رایج‌تر را بررسی خواهد کرد و توضیح خواهد داد که چگونه کار می‌کنند و چگونه می‌توان آنها را با کد مدرن ادغام کرد.

</div>



<div dir="rtl" align="right">

گاهی اوقات، یک نوع خاص در طول سال‌ها به‌روز می‌شود و اعضای بیشتری را کسب می‌کند در حالی که از الگوهای ناهمزمان مختلفی پشتیبانی می‌کند. شاید بهترین مثال برای این موضوع کلاس `Socket` باشد. در اینجا برخی از اعضای کلاس `Socket` برای عملیات اصلی `Send` آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس Socket
{
    // همزمان
    public int Send(byte[] buffer, int offset, int size,
        SocketFlags flags);

    // APM
    public IAsyncResult BeginSend(byte[] buffer, int offset, int
        size,
        SocketFlags flags, AsyncCallback callback, object state);
    public int EndSend(IAsyncResult result);

    // سفارشی، بسیار نزدیک به APM
    public IAsyncResult BeginSend(byte[] buffer, int offset, int
        size,
        SocketFlags flags, out SocketError error,
        AsyncCallback callback, object state);
    public int EndSend(IAsyncResult result, out SocketError
        error);

    // سفارشی
    public bool SendAsync(SocketAsyncEventArgs e);

    // TAP (به عنوان یک متد توسعه)
    public Task<int> SendAsync(ArraySegment<byte> buffer,
        SocketFlags socketFlags);

    // TAP (به عنوان یک متد توسعه) با استفاده از انواع کارآمدتر
    public ValueTask<int> SendAsync(ReadOnlyMemory<byte> buffer,
        SocketFlags socketFlags, CancellationToken
        cancellationToken = default);
}
```

</div>

<div dir="rtl" align="right">

متأسفانه، با اکثر مستندسازی‌های الفبایی و تعداد زیادی از overloadها در تلاش برای ساده‌سازی استفاده، انواعی مانند `Socket` درک آنها دشوار می‌شود. امیدواریم راهنماهای این بخش به شما کمک کند.

</div>

<div dir="rtl" align="right">

## الگوی ناهمزمان مبتنی بر وظیفه (TAP)

الگوی ناهمزمان مبتنی بر وظیفه (TAP) الگوی API ناهمزمان مدرنی است که برای استفاده با `await` آماده است. هر عملیات ناهمزمان با یک متد واحد نشان داده می‌شود که یک awaitable را برمی‌گرداند. یک "awaitable" هر نوعی است که می‌تواند توسط `await` مصرف شود؛ این معمولاً `Task` یا `Task<T>` است اما ممکن است `ValueTask`، `ValueTask<T>`، نوعی که توسط یک چارچوب تعریف شده (مثلاً `IAsyncAction` یا `IAsyncOperation<T>`، که توسط برنامه‌های جهانی ویندوز استفاده می‌شود)، یا حتی یک نوع سفارشی تعریف شده توسط یک کتابخانه باشد.

معمولاً متدهای TAP دارای پسوند `Async` هستند. با این حال، این فقط یک قرارداد است؛ همه متدهای TAP لزوماً پسوند `Async` ندارند. این پسوند می‌تواند حذف شود اگر توسعه‌دهنده API معتقد باشد که زمینه ناهمزمان به اندازه کافی مشخص است؛ برای مثال، `Task.WhenAll` و `Task.WhenAny` پسوند `Async` ندارند. علاوه بر این، توجه داشته باشید که پسوند `Async` ممکن است در متدهای غیر TAP نیز وجود داشته باشد (مثلاً `WebClient.DownloadStringAsync` یک متد TAP نیست). الگوی معمول در این مورد این است که متد TAP پسوند `TaskAsync` داشته باشد (مثلاً `WebClient.DownloadStringTaskAsync` یک متد TAP است).

متدهایی که جریان‌های ناهمزمان را برمی‌گردانند نیز از یک الگوی مشابه TAP پیروی می‌کنند، با استفاده از `Async` به عنوان پسوند. اگرچه آنها awaitable برنمی‌گردانند، اما جریان‌های awaitable برمی‌گردانند - انواعی که می‌توان با استفاده از `await foreach` آنها را مصرف کرد.

الگوی ناهمزمان مبتنی بر وظیفه را می‌توان با این مشخصات شناخت:
۱. عملیات با یک متد واحد نشان داده می‌شود.
۲. متد یک awaitable یا یک جریان awaitable را برمی‌گرداند.
۳. متد معمولاً با `Async` به پایان می‌رسد.

در اینجا مثالی از یک نوع با API TAP آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس ExampleHttpClient
{
    public Task<string> GetStringAsync(Uri requestUri);

    // معادل همزمان، برای مقایسه
    public string GetString(Uri requestUri);
}
```

</div>

<div dir="rtl" align="right">

مصرف الگوی ناهمزمان مبتنی بر وظیفه با استفاده از `await` انجام می‌شود و بخش‌های بزرگی از این کتاب به آن اختصاص دارد. اگر به طور اتفاقی به این ضمیمه رسیده‌اید بدون اینکه بدانید چگونه از `await` استفاده کنید، فکر نمی‌کنم بتوانم در این مرحله کمکی به شما کنم، اما می‌توانید فصل‌های ۱ و ۲ را بخوانید تا ببینید آیا حافظه‌تان تحریک می‌شود یا خیر.

</div>

<div dir="rtl" align="right">

## مدل برنامه‌نویسی ناهمزمان (APM)

پس از TAP، الگوی مدل برنامه‌نویسی ناهمزمان (APM) احتمالاً رایج‌ترین الگویی است که با آن مواجه خواهید شد. این اولین الگویی بود که در آن عملیات ناهمزمان دارای نمایندگان شیء درجه اول بودند. نشانه اصلی این الگو، اشیاء `IAsyncResult` همراه با یک جفت متد است که عملیات را مدیریت می‌کنند، یکی با `Begin` شروع می‌شود و دیگری با `End`.

`IAsyncResult` به شدت تحت تأثیر I/O هم‌پوشانی بومی قرار داشت.

الگوی APM اجازه می‌دهد که کد مصرف‌کننده به صورت همزمان یا ناهمزمان رفتار کند. کد مصرف‌کننده می‌تواند از این گزینه‌ها استفاده کند:
*   انسداد برای تکمیل عملیات. این کار با فراخوانی متد `End` انجام می‌شود.
*   نظرسنجی برای تکمیل عملیات در حین انجام کار دیگر.
*   ارائه یک نماینده فراخوانی برای اجرا زمانی که عملیات تکمیل می‌شود.

</div>


<div dir="rtl" align="right">

در تمام موارد، کد مصرف‌کننده در نهایت باید متد `End` را فراخوانی کند تا نتایج عملیات ناهمزمان را دریافت کند. اگر عملیات زمانی که `End` فراخوانی می‌شود تکمیل نشده باشد، رشته فراخوانی‌کننده مسدود خواهد شد تا عملیات تکمیل شود.

متد `Begin` یک پارامتر `AsyncCallback` و یک پارامتر شیء (معمولاً `state` نامیده می‌شود) به عنوان دو پارامتر آخر دارد. این‌ها توسط کد مصرف‌کننده برای ارائه یک نماینده فراخوانی استفاده می‌شوند که زمانی که عملیات تکمیل می‌شود اجرا می‌گردد. پارامتر شیء می‌تواند هر چیزی باشد؛ این یک میراث از روزهای بسیار اولیه .NET است، قبل از وجود متدهای لامبدا یا حتی متدهای ناشناس. فقط برای ارائه زمینه به پارامتر `AsyncCallback` استفاده می‌شود.

APM در کتابخانه‌های مایکروسافت بسیار گسترده است، اما در اکوسیستم گسترده‌تر .NET چندان رایج نیست. این به دلیل آن است که هرگز پیاده‌سازی‌های `IAsyncResult` برای استفاده مجدد در دسترس قرار نگرفت و پیاده‌سازی صحیح این رابط نسبتاً پیچیده است. علاوه بر این، ترکیب سیستم‌های مبتنی بر APM دشوار است. من فقط چند پیاده‌سازی سفارشی `IAsyncResult` در دنیای واقعی دیده‌ام؛ همه آنها نسخه‌ای از پیاده‌سازی عمومی `IAsyncResult` جفری ریچتر بودند، که در مقاله‌اش با عنوان "امور همزمان: پیاده‌سازی مدل برنامه‌نویسی ناهمزمان CLR" در شماره مارس ۲۰۰۷ مجله MSDN منتشر شد.

الگوی مدل برنامه‌نویم ناهمزمان را می‌توان با این مشخصات شناخت:
۱. عملیات با یک جفت متد نشان داده می‌شود، یکی با `Begin` شروع می‌شود و دیگری با `End`.
۲. متد `Begin` یک `IAsyncResult` برمی‌گرداند و تمام پارامترهای ورودی معمولی را همراه با یک پارامتر `AsyncCallback` اضافی و یک پارامتر شیء اضافی دریافت می‌کند.
۳. متد `End` فقط یک `IAsyncResult` را دریافت می‌کند و مقدار نتیجه را در صورت وجود برمی‌گرداند.

در اینجا مثالی از یک نوع با API APM آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس MyHttpClient
{
    public IAsyncResult BeginGetString(Uri requestUri,
        AsyncCallback callback, object state);
    public string EndGetString(IAsyncResult asyncResult);

    // معادل همزمان، برای مقایسه
    public string GetString(Uri requestUri);
}
```

</div>

<div dir="rtl" align="right">

APM را با استفاده از `Task.Factory.FromAsync` به TAP تبدیل کنید؛ به دستور ۸.۲ و مستندات مایکروسافت مراجعه کنید.

مواردی وجود دارد که کد تقریباً از الگوی APM پیروی می‌کند، اما نه دقیقاً؛ به عنوان مثال، کتابخانه‌های قدیمی مایکروسافت TeamFoundation پارامتر شیء را در متدهای `Begin` خود شامل نمی‌کردند. در این موارد، `Task.Factory.FromAsync` کار نخواهد کرد و شما دو گزینه خواهید داشت. گزینه کم‌کارآمدتر این است که متد `Begin` را فراخوانی کنید و `IAsyncResult` را به `FromAsync` منتقل کنید. گزینه کمتر زیبا استفاده از `TaskCompletionSource<T>` با قابلیت انعطاف بیشتر است؛ به دستور ۸.۳ مراجعه کنید.

</div>

<div dir="rtl" align="right">

## برنامه‌نویسی ناهمزمان مبتنی بر رویداد (EAP)

برنامه‌نویسی ناهمزمان مبتنی بر رویداد (EAP) یک جفت متد/رویداد مطابق را تعریف می‌کند. متد معمولاً با `Async` به پایان می‌رسد و در نهایت باعث ایجاد رویدادی می‌شود که با `Completed` پایان می‌یابد.

چند نکته احتیاطی در کار با EAP وجود دارد که آن را کمی پیچیده‌تر از آنچه در ابتدا به نظر می‌رسد می‌کند. اولاً، باید به خاطر داشته باشید که قبل از فراخوانی متد، کنترل‌کننده خود را به رویداد اضافه کنید؛ در غیر این صورت، شرایط مسابقه‌ای خواهید داشت که در آن رویداد ممکن است قبل از اشتراک شما رخ دهد و هرگز آن را کامل نبینید. ثانیاً، اجزای نوشته شده در الگوی EAP معمولاً در نقطه‌ای `SynchronizationContext` فعلی را ضبط می‌کنند و سپس رویداد خود را در آن زمینه ایجاد می‌کنند. برخی از اجزا `SynchronizationContext` را در سازنده ضبط می‌کنند و برخی دیگر آن را در زمان فراخوانی متد و شروع عملیات ناهمزمان ضبط می‌کنند.

الگوی برنامه‌نویسی ناهمزمان مبتنی بر رویداد را می‌توان با این مشخصات شناخت:
۱. عملیات با یک رویداد و یک متد نشان داده می‌شود.
۲. رویداد با `Completed` پایان می‌یابد.
۳. نوع رویداد args برای رویداد `Completed` ممکن است از `AsyncCompletedEventArgs` ارث برده باشد.
۴. متد معمولاً با `Async` پایان می‌یابد.
۵. متد `void` را برمی‌گرداند.

متدهای EAP که با `Async` پایان می‌یابند از متدهای TAP که با `Async` پایان می‌یابند قابل تشخیص هستند زیرا متدهای EAP `void` را برمی‌گردانند، در حالی که متدهای TAP یک نوع `awaitable` را برمی‌گردانند.

در اینجا مثالی از یک نوع با API EAP آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس GetStringCompletedEventArgs : AsyncCompletedEventArgs
{
    public string Result { get; }
}

کلاس MyHttpClient
{
    public void GetStringAsync(Uri requestUri);
    public event Action<object, GetStringCompletedEventArgs>
        GetStringCompleted;

    // معادل همزمان، برای مقایسه
    public string GetString(Uri requestUri);
}
```

</div>

<div dir="rtl" align="right">

EAP را با استفاده از `TaskCompletionSource<T>` به TAP تبدیل کنید؛ به دستور ۸.۳ و مستندات مایکروسافت مراجعه کنید.

</div>

<div dir="rtl" align="right">

## سبک انتقال ادامه (CPS)

این الگویی است که در زبان‌های دیگر، به‌ویژه جاوااسکریپت و TypeScript که توسط توسعه‌دهندگان Node.js استفاده می‌شود، بسیار رایج‌تر است. در این الگو، هر عملیات ناهمزمان یک نماینده فراخوانی را دریافت می‌کند که زمانی که عملیات با موفقیت یا با خطا تکمیل می‌شود، فراخوانی می‌گردد. یک نسخه از این الگو از دو نماینده فراخوانی استفاده می‌کند، یکی برای موفقیت و دیگری برای خطا. این نوع فراخوانی "ادامه" نامیده می‌شود و ادامه به عنوان یک پارامتر منتقل می‌شود، از همین رو نام "سبک انتقال ادامه" آمده است. این الگو هرگز در دنیای .NET رایج نبود، اما چند کتابخانه متن‌باز قدیمی وجود دارد که از آن استفاده کرده‌اند.

</div>



<div dir="rtl" align="right">

الگوی سبک انتقال ادامه را می‌توان با این مشخصات شناخت:
۱. عملیات با یک متد واحد نشان داده می‌شود.
۲. متد یک پارامتر اضافی که یک نماینده فراخوانی است دریافت می‌کند؛ نماینده فراخوانی دو آرگومان دارد، یکی برای خطاها و دیگری برای نتایج.
۳. به طور جایگزین، متد عملیات دو پارامتر اضافی دریافت می‌کند، هر دو نماینده فراخوانی؛ یک نماینده فراخوانی فقط برای خطاها و نماینده دیگر فقط برای نتایج.
۴. نماینده‌های فراخوانی معمولاً `done` یا `next` نامیده می‌شوند.

در اینجا مثالی از یک نوع با API سبک انتقال ادامه آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس MyHttpClient
{
    public void GetString(Uri requestUri, Action<Exception,
        string> done);

    // معادل همزمان، برای مقایسه
    public string GetString(Uri requestUri);
}
```

</div>

<div dir="rtl" align="right">

CPS را با استفاده از `TaskCompletionSource<T>` به TAP تبدیل کنید، با انتقال نماینده‌های فراخوانی که فقط `TaskCompletionSource<T>` را تکمیل می‌کنند؛ به دستور ۸.۳ مراجعه کنید.

</div>

<div dir="rtl" align="right">

## الگوهای ناهمزمان سفارشی

انواع بسیار تخصصی گاهی اوقات الگوهای ناهمزمان خود را تعریف می‌کنند. مشهورترین مثال از این نوع، نوع `Socket` است که الگویی را تعریف کرد که نمونه‌های `SocketAsyncEventArgs` را که نشان‌دهنده عملیات هستند، منتقل می‌کند. دلیل معرفی این الگو این بود که `SocketAsyncEventArgs` می‌تواند مجدداً استفاده شود و در نتیجه چرخش حافظه برای برنامه‌هایی که فعالیت شبکه سنگینی دارند کاهش یابد. برنامه‌های مدرن می‌توانند از `ValueTask<T>` با `ManualResetValueTaskSourceCore<T>` برای دستیابی به چنین افزایش‌های عملکردی مشابه استفاده کنند.

الگوهای سفارشی هیچ مشخصه مشترکی ندارند و بنابراین سخت‌ترین برای شناسایی هستند. خوشبختانه، الگوهای ناهمزمان سفارشی نادر هستند.

در اینجا مثالی از یک نوع با API ناهمزمان سفارشی آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس MyHttpClient
{
    public void GetString(Uri requestUri,
        MyHttpClientAsynchronousOperation operation);

    // معادل همزمان، برای مقایسه
    public string GetString(Uri requestUri);
}
```

</div>

<div dir="rtl" align="right">

`TaskCompletionSource<T>` تنها راه برای مصرف الگوهای ناهمزمان سفارشی است؛ به دستور ۸.۳ مراجعه کنید.

</div>

<div dir="rtl" align="right">

## ISynchronizeInvoke

تمام الگوهای قبلی برای عملیات ناهمزمانی هستند که شروع می‌شوند و پس از شروع، یک بار تکمیل می‌شوند. برخی از اجزا از مدل اشتراک پیروی می‌کنند: آنها یک جریان رویدادهای مبتنی بر Push را نشان می‌دهند، نه یک عملیات واحد که یک بار شروع و تکمیل می‌شود. یک مثال خوب از مدل اشتراک، نوع `FileSystemWatcher` است. برای مشاهده تغییرات سیستم فایل، کد مصرف‌کننده ابتدا به چندین رویداد مشترک می‌شود و سپس ویژگی `EnableRaisingEvents` را روی `true` تنظیم می‌کند. زمانی که `EnableRaisingEvents` true باشد، ممکن است چندین رویداد تغییر سیستم فایل ایجاد شود.

برخی از اجزا از الگوی `ISynchronizeInvoke` برای رویدادهای خود استفاده می‌کنند. آنها یک ویژگی `ISynchronizeInvoke` را ارائه می‌دهند و مصرف‌کنندگان آن را به یک پیاده‌سازی تنظیم می‌کنند که به جزء اجازه می‌دهد کار را زمان‌بندی کند. این معمولاً برای زمان‌بندی کار در رشته رابط کاربری استفاده می‌شود تا رویدادهای جزء در رشته رابط کاربری ایجاد شوند. طبق قرارداد، اگر `ISynchronizeInvoke` null باشد، هیچ همگام‌سازی رویدادها انجام نمی‌شود و ممکن است در رشته‌های پس‌زمینه ایجاد شوند.

الگوی `ISynchronizeInvoke` را می‌توان با این مشخصات شناخت:
۱. یک ویژگی از نوع `ISynchronizeInvoke` وجود دارد.
۲. ویژگی معمولاً `SynchronizingObject` نامیده می‌شود.

در اینجا مثالی از یک نوع که از الگوی `ISynchronizeInvoke` استفاده می‌کند آورده شده است:

</div>

<div dir="ltr" align="left">

```csharp
کلاس MyHttpClient
{
    public ISynchronizeInvoke SynchronizingObject { get; set; }
    public void StartListening();
    public event Action<string> StringArrived;
}
```

</div>

<div dir="rtl" align="right">

از آنجا که `ISynchronizeInvoke` مستلزم چندین رویداد در یک مدل اشتراک است، راه صحیح برای مصرف این اجزا، تبدیل این رویدادها به یک جریان قابل مشاهده است، با استفاده از `FromEvent` (به دستور ۶.۱ مراجعه کنید) یا `Observable.Create`.

</div>
