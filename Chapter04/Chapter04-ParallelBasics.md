

<div dir="rtl" align="right">

# فصل ۴. اصول اولیه موازی‌سازی

این فصل به بررسی الگوهای برنامه‌نویسی موازی می‌پردازد.  
برنامه‌نویسی موازی برای تقسیم وظایف پردازنده‌محور (CPU-bound) به بخش‌های کوچک‌تر و اجرای آن‌ها در میان چندین نخ (thread) کاربرد دارد.

> توجه: دستورالعمل‌های این فصل فقط برای کارهای CPU-bound مناسب هستند.  
اگر با عملیاتی ذاتاً غیرهمزمان (مانند ورودی/خروجی یا I/O-bound) سروکار دارید که می‌خواهید به صورت موازی اجرا شوند، به فصل ۲ (به‌ویژه دستورالعمل ۲.۴) مراجعه نمایید.

ابسترکشن‌های پردازش موازی بررسی‌شده در این فصل، بخشی از «کتابخانه TPL» (Task Parallel Library) در چارچوب دات‌نت (.NET) هستند.

---

## ۴.۱ - پردازش موازی داده‌ها

### مسئله

فرض کنید مجموعه‌ای داده دارید و باید عملی یکسان روی هر عنصر آن انجام دهید. این عمل پردازنده‌محور بوده و انجام آن زمان‌بر است.

### راه‌حل

نوع `Parallel` متدی به نام `ForEach` دارد که دقیقاً برای این سناریو طراحی شده است. مثال زیر مجموعه‌ای از ماتریس‌ها را در نظر گرفته و همه را می‌چرخاند:

<div dir="ltr" align="left">

```csharp
void RotateMatrices(IEnumerable<Matrix> matrices, float degrees)
{
    Parallel.ForEach(matrices, matrix => matrix.Rotate(degrees));
}
```
</div>

برخی اوقات لازم است حلقه را زودتر متوقف کنید، مثلاً اگر به یک مقدار نامعتبر برخورد شد. مثال زیر هر ماتریس را معکوس می‌کند، ولی در صورت یافتن ماتریس نامعتبر، حلقه را خاتمه می‌دهد:

<div dir="ltr" align="left">

```csharp
void InvertMatrices(IEnumerable<Matrix> matrices)
{
    Parallel.ForEach(matrices, (matrix, state) =>
    {
        if (!matrix.IsInvertible)
            state.Stop();
        else
            matrix.Invert();
    });
}
```
</div>

در کد بالا از `ParallelLoopState.Stop` استفاده شده است تا حلقه را متوقف کرده و مانع اجرای بدنه حلقه برای عناصر بعدی شود.

> دقت کنید: این یک حلقه موازی است؛ بنابراین، ممکن است اجرای بدنه حلقه برای برخی عناصر (حتی عناصر بعدی در دنباله) همزمان انجام شده باشد. به بیان دیگر، اگر مثلاً سومین ماتریس قابل معکوس‌سازی نباشد، حلقه متوقف می‌شود و ماتریس جدیدی پردازش نمی‌گردد؛ با این حال، ممکن است ماتریس‌های چهارم یا پنجم در حال پردازش باشند.

وضعیتی رایج‌تر، نیاز به لغو (cancel) یک حلقه موازی توسط کد خارجی است؛ یعنی توقف از بیرون حلقه، نه درون آن.  
برای مثال، یک دکمه انصراف می‌تواند باعث لغو یک CancellationTokenSource و توقف حلقه شود:

<div dir="ltr" align="left">

```csharp
void RotateMatrices(IEnumerable<Matrix> matrices, float degrees, CancellationToken token)
{
    Parallel.ForEach(
        matrices,
        new ParallelOptions { CancellationToken = token },
        matrix => matrix.Rotate(degrees)
    );
}
```
</div>

### نکته امنیت داده

هر کار موازی می‌تواند روی نخ‌های متفاوت اجرا شود؛ بنابراین وضعیت‌های مشترک باید ایمن نگه‌داشته شوند.  
مثال زیر تعداد ماتریس‌های غیرقابل معکوس‌سازی را محاسبه می‌کند و از `lock`‌ برای جلوگیری از رقابت استفاده می‌کند:

<div dir="ltr" align="left">

```csharp
// توجه: این مورد بهینه‌ترین شیوه نیست و صرفاً برای شرح موضوع است.
int InvertMatrices(IEnumerable<Matrix> matrices)
{
    object mutex = new object();
    int nonInvertibleCount = 0;
    Parallel.ForEach(matrices, matrix =>
    {
        if (matrix.IsInvertible)
        {
            matrix.Invert();
        }
        else
        {
            lock (mutex)
            {
                ++nonInvertibleCount;
            }
        }
    });
    return nonInvertibleCount;
}
```
</div>

---

### بحث و نکات تکمیلی


- متد `Parallel.ForEach` امکان پردازش موازی یک دنباله مقادیر را فراهم می‌کند.
- روش مشابه دیگری به نام Parallel LINQ یا PLINQ وجود دارد که قابلیت‌های تقریبا مشابه‌ای با نگارش LINQ فراهم می‌کند.
- تفاوت مهم:  
 PLINQ فرض می‌کند که می‌تواند از **تمام هسته‌های سیستم** استفاده کند.  
- ولی `Parallel` به شرایط لحظه‌ای CPU واکنش پویا نشان می‌دهد.
- متد `Parallel.ForEach` در واقع یک حلقه foreach موازی است. اگر به حلقه for موازی نیاز دارید، متد `Parallel.For` نیز وجود دارد.
 `Parallel.For` مخصوص زمانی است که چندین آرایه داده دارید که باید همگی مؤلفه‌هایی با ایندکس یکسان رویشان پردازش شود.


#### همچنین ببینید

- دستورالعمل ۴.۲: جمع‌بندی موازی (جمع، میانگین، ...)
- دستورالعمل ۴.۵: اصول اولیه PLINQ
- فصل ۱۰: لغو (Cancellation)

</div>




<div dir="rtl" align="right">

## ۴.۲ - تجمیع موازی (Parallel Aggregation)

### مسئله

در پایان یک عملیات موازی، لازم است نتایج را تجمیع (aggregate) کنید.  
برای مثال: جمع کردن همه مقادیر یا محاسبه میانگین آن‌ها.

### راه‌حل

کلاس `Parallel` از تجمیع داده‌ها به روش مقادیر محلی (local values) پشتیبانی می‌کند؛  
یعنی در هر بخش حلقه موازی، یک متغیر محلی وجود دارد که بار اصلی محاسبات همان قسمت را به عهده می‌گیرد.  
بدین ترتیب، بدنه حلقه می‌تواند به صورت بی‌نیاز از هماهنگ‌سازی (synchronization) به مقدار محلی دسترسی داشته باشد.

زمانی که حلقه آماده تجمیع نتایج محلی شد، این کار با delegate مخصوصی به نام **localFinally** انجام می‌شود.  
دسترسی و ترکیب نتایج نهایی (global state) ــ معمولاً در بخش localFinally ــ نیازمند هماهنگ‌سازی (مثلاً به کمک lock) است.

#### نمونه جمع موازی با Parallel.ForEach

<div dir="ltr" align="left">

```csharp
// توجه: این روش بهینه‌ترین نیست و صرفاً برای نمایش عملکرد lock در محافظت وضعیت مشترک است.
int ParallelSum(IEnumerable<int> values)
{
    object mutex = new object();
    int result = 0;
    Parallel.ForEach(
        source: values,
        localInit: () => 0,
        body: (item, state, localValue) => localValue + item,
        localFinally: localValue =>
        {
            lock (mutex)
                result += localValue;
        }
    );
    return result;
}
```
</div>

#### جمع موازی با PLINQ

PLINQ (Parallel LINQ) پشتیبانی ساده‌تر و طبیعی‌تری برای تجمیع فراهم کرده است.  
مثال ساده جمع عددی با PLINQ را ببینید:

<div dir="ltr" align="left">

```csharp
int ParallelSum(IEnumerable<int> values)
{
    return values.AsParallel().Sum();
}
```
</div>

این مثال برای عملگرهای رایج (مثلاً جمع) کاملاً سرراست است،  
زیرا PLINQ به طور پیش‌فرض از آن‌ها پشتیبانی می‌کند.

#### تجمیع عمومی‌تر با PLINQ و عملگر Aggregate

PLINQ همچنین امکان تجمیع‌های پیچیده‌تر را با استفاده از عملگر `Aggregate` فراهم می‌کند:

<div dir="ltr" align="left">

```csharp
int ParallelSum(IEnumerable<int> values)
{
    return values.AsParallel().Aggregate(
        seed: 0,
        func: (sum, item) => sum + item
    );
}
```
</div>

---

### بحث و نکات تکمیلی

- اگر قبلاً در پروژه خود از کلاس `Parallel` استفاده می‌کنید، می‌توانید از قابلیت تجمیع داخلی آن بهره ببرید.
- در غیر این صورت، **PLINQ** معمولاً انتخاب خواناتر، کوتاه‌تر و صریح‌تری است و پیشنهاد می‌شود.

#### همچنین ببینید

- دستورالعمل ۴.۵ اصول اولیه PLINQ را توضیح می‌دهد.

</div>




<div dir="rtl" align="right">

## ۴.۳ - فراخوانی موازی (Parallel Invocation)

### مسئله

فرض کنید چندین متد مستقل دارید که باید همزمان (به صورت موازی) اجرا شوند و این متدها عمدتاً ارتباطی با یکدیگر ندارند.

### راه‌حل

کلاس `Parallel` یک عضو ساده و مؤثر به نام `Invoke` دارد که برای این گونه سناریوها طراحی شده است.

در مثال زیر، یک آرایه به دو بخش تقسیم می‌شود و هریک به شکل جداگانه ــ اما همزمان ــ پردازش می‌گردد:

<div dir="ltr" align="left">

```csharp
void ProcessArray(double[] array)
{
    Parallel.Invoke(
        () => ProcessPartialArray(array, 0, array.Length / 2),
        () => ProcessPartialArray(array, array.Length / 2, array.Length)
    );
}

void ProcessPartialArray(double[] array, int begin, int end)
{
    // پردازش زمان‌بر از لحاظ پردازنده...
}
```
</div>

اگر تعداد فراخوانی‌ها (delegate‌ها) در زمان اجرا معلوم می‌شود، می‌توانید یک آرایه از delegateها به متد `Parallel.Invoke` بدهید:

<div dir="ltr" align="left">

```csharp
void DoAction20Times(Action action)
{
    Action[] actions = Enumerable.Repeat(action, 20).ToArray();
    Parallel.Invoke(actions);
}
```
</div>

`Parallel.Invoke` مانند سایر اعضای خانواده Parallel، از قابلیت لغو شدن (cancellation) نیز پشتیبانی می‌کند:

<div dir="ltr" align="left">

```csharp
void DoAction20Times(Action action, CancellationToken token)
{
    Action[] actions = Enumerable.Repeat(action, 20).ToArray();
    Parallel.Invoke(new ParallelOptions { CancellationToken = token }, actions);
}
```
</div>

---

### بحث و نکات تکمیلی

 `Parallel.Invoke` بسیار مناسب برای فراخوانی همزمان مجموعه‌ای از متدهای مستقل با منطق متفاوت است.
- دقت کنید:
  - اگر لازم است روی مجموعه داده‌ای یک عملیات مشابه به تعداد دفعات بالا اجرا کنید، `Parallel.ForEach` گزینه بهتری است.
  - اگر هر عملیات خروجی قابل برگشتی دارد، اغلب بهتر است از **PLINQ (Parallel LINQ)** استفاده نمایید.

#### همچنین ببینید

- دستورالعمل ۴.۱: درباره‌ی Parallel.ForEach و پردازش موازی داده‌ها.
- دستورالعمل ۴.۵: اصول اولیه Parallel LINQ (PLINQ).

</div>





<div dir="rtl" align="right">

## ۴.۴ - موازی‌سازی پویا (Dynamic Parallelism)

### مسئله

در مواقعی ساختار و تعداد کارهای موازی تنها در زمان اجرا (runtime) تعیین می‌شود و نمی‌توان از الگوهای ساده‌تر مانند Parallel.ForEach یا Invoke استفاده کرد.

### راه‌حل

کتابخانه وظایف موازی (Task Parallel Library – TPL) مبتنی بر نوع داده‌ی **Task** طراحی شده است.  
کلاس‌های `Parallel` و `PLINQ` صرفاً میانبرهای سطح بالایی برای قابلیت‌های قدرتمند Task به حساب می‌آیند.  
برای موازی‌سازی پویا، بهترین راه استفاده مستقیم از Task است.

در مثال زیر باید عملی زمان‌بر روی هر گره‌ی یک درخت دودویی انجام شود.  
ساختار درخت هنگام اجرا مشخص می‌شود و هر گره باید والد خودش را قبل از فرزندان پردازش کند.  
بنا بر این، این سناریو مصداق بارز موازی‌سازی پویاست.

متد Traverse گره فعلی را پردازش کرده و برای هر زیرشاخه (در صورت وجود) یک Task فرزند ایجاد می‌کند:

<div dir="ltr" align="left">

```csharp
void Traverse(Node current)
{
    DoExpensiveActionOnNode(current);
    if (current.Left != null)
    {
        Task.Factory.StartNew(
            () => Traverse(current.Left),
            CancellationToken.None,
            TaskCreationOptions.AttachedToParent,
            TaskScheduler.Default);
    }
    if (current.Right != null)
    {
        Task.Factory.StartNew(
            () => Traverse(current.Right),
            CancellationToken.None,
            TaskCreationOptions.AttachedToParent,
            TaskScheduler.Default);
    }
}

void ProcessTree(Node root)
{
    Task task = Task.Factory.StartNew(
        () => Traverse(root),
        CancellationToken.None,
        TaskCreationOptions.None,
        TaskScheduler.Default);
    task.Wait();
}
```
</div>

پرچم `AttachedToParent` تضمین می‌کند Task هر شاخه به والدش متصل است،  
در نتیجه رابطه والد/فرزند بین Taskها دقیقاً مطابق با ساختار درخت شکل می‌گیرد.  
Task والد ابتدا delegate خود را اجرا می‌کند و سپس تا اتمام Taskهای فرزند خود منتظر می‌ماند.  
اگر در تسک‌های فرزند Exception رخ دهد، به والد منتقل می‌شود. بنابراین  
کافی است روی Task ریشه Wait کنید تا مطمئن شوید تمامی تسک‌های مربوط به هر گره اجرا شده‌اند.

#### سناریوهای بدون والد/فرزند

اگر ساختار شما والد/فرزند ندارد، می‌توانید **اجراها را با continuation** به یکدیگر متصل کنید.  
Continuation یک Task مجزا است که پس از پایان Task قبلی اجرا می‌شود:

<div dir="ltr" align="left">

```csharp
Task task = Task.Factory.StartNew(
    () => Thread.Sleep(TimeSpan.FromSeconds(2)),
    CancellationToken.None,
    TaskCreationOptions.None,
    TaskScheduler.Default);

Task continuation = task.ContinueWith(
    t => Trace.WriteLine("Task is done"),
    CancellationToken.None,
    TaskContinuationOptions.None,
    TaskScheduler.Default);
// آرگومان "t" در continuation همان Task اولیه است.
```
</div>

---

### بحث و نکات تکمیلی

- در مثال بالا از `CancellationToken.None` و `TaskScheduler.Default` استفاده شد.  
  توکن‌های لغو در دستورالعمل ۱۰.۲ و برنامه‌ریزهای تسک در دستورالعمل ۱۳.۳ توضیح داده شده‌اند.
- همیشه بهتر است TaskScheduler را به‌صراحت هنگام استفاده از StartNew یا ContinueWith مشخص کنید.
- ساختار والد/فرزند میان تسک‌ها، در موازی‌سازی پویا رایج است اما الزامی نیست.
- می‌توانید همه Taskهای موازی را در یک مجموعه thread-safe جمع‌آوری و در نهایت با `Task.WaitAll` منتظر پایان همگی بمانید.

#### هشدار مهم

استفاده از Task برای "پردازش موازی" کاملاً متفاوت از استفاده از Task برای "پردازش غیرهمزمان" (asynchronous) است:

- **تسک‌های موازی**:
    - معمولاً با متدهای مسدودکننده مانند `Task.Wait`، `Task.Result`، `Task.WaitAll`، `Task.WaitAny` و ساختار AttachedToParent کار می‌کنند.
    - باید با `Task.Run` یا `Task.Factory.StartNew` ساخته شوند.
- **تسک‌های غیرهمزمان**:
    - باید از الگوهای غیرمسدودکننده مانند `await`، `Task.WhenAll` و `Task.WhenAny` بهره ببرند.
    - از AttachedToParent استفاده نمی‌کنند و رابطه والد/فرزند، ضمنی از طریق await ایجاد می‌گردد.

---

#### همچنین ببینید

- دستورالعمل ۴.۳: درباره‌ی اجرای موازی مجموعه‌ای از متدهای مشخص و مستقل.
  
</div>





<div dir="rtl" align="right">

## ۴.۵ - Parallel LINQ (PLINQ)

### مسئله

گاهی نیاز دارید پردازش موازی روی یک دنباله داده انجام دهید؛  
هدف می‌تواند تولید یک دنباله جدید یا استخراج خلاصه‌ای از داده (مانند جمع یا میانگین) باشد.

---

### راه‌حل

بیشتر توسعه‌دهندگان با LINQ آشنایی دارند و از آن برای نگارش محاسبات مبتنی بر pull روی داده‌ها استفاده می‌کنند.  
Parallel LINQ (PLINQ) این امکانات را با قابلیت پردازش موازی، گسترش می‌دهد.

PLINQ به‌ویژه برای سناریوهای **جریان داده‌ای (streaming)** مناسب است؛  
زمانی که ورودی شما یک دنباله است و خروجی نهایی نیز به شکل یک دنباله تولید می‌شود.

#### مثال: ضرب هر عنصر در ۲ به شکل موازی

<div dir="ltr" align="left">

```csharp
IEnumerable<int> MultiplyBy2(IEnumerable<int> values)
{
    return values.AsParallel().Select(value => value * 2);
}
```
</div>

> **نکته:**  
> خروجی PLINQ لزوماً با ترتیب اولیه دنباله ورودی یکی نیست. این رفتار پیش‌فرض PLINQ است.

در صورت نیاز به حفظ ترتیب ورودی اصلی، می‌توانید از `.AsOrdered()` در زنجیره PLINQ استفاده کنید:

<div dir="ltr" align="left">

```csharp
IEnumerable<int> MultiplyBy2(IEnumerable<int> values)
{
    return values.AsParallel().AsOrdered().Select(value => value * 2);
}
```
</div>

---

#### تجمیع موازی با PLINQ

یکی دیگر از کاربردهای شناخته‌شده PLINQ، تجمیع یا خلاصه‌سازی داده‌ها به شکل موازی است. مثال:

<div dir="ltr" align="left">

```csharp
int ParallelSum(IEnumerable<int> values)
{
    return values.AsParallel().Sum();
}
```
</div>

---

### بحث و نکات تکمیلی

- کلاس `Parallel` برای بسیاری از سناریوها مناسب است؛ اما برای تجمیع یا تبدیل یک دنباله به دنباله دیگر، PLINQ معمولا ساده‌تر و خواناتر است.
- توجه داشته باشید:  
  کلاس Parallel نسبت به منابع سیستم دوستانه‌تر (با رعایت Fairness) عمل می‌کند، خصوصا هنگام اجرا روی سرور؛ PLINQ معمولاً تا حد نهایی منابع را صرف محاسبات موازی می‌کند.
 PLINQ نسخه‌های موازی طیف وسیعی از عملگرهای LINQ را پشتیبانی می‌کند، مانند:
  - فیلتر کردن (`Where`)
  - نگاشت/پروجکشن (`Select`)
  - تجمیع مانند `Sum`، `Average` و حتی `Aggregate` 
- به طور کلی، هر عملی که با LINQ بتوانید انجام دهید، با PLINQ قابل انجام است؛ تنها کافیست دنباله را با `.AsParallel()` تبدیل کنید.
- این قابلیت، PLINQ را انتخابی عالی می‌کند؛ به‌خصوص اگر کد LINQ موجودی دارید که با موازی‌سازی، سرعت خواهد گرفت.

---

#### همچنین ببینید

- **دستورالعمل ۴.۱**: نحوه استفاده از کلاس `Parallel` برای پردازش موازی هر عنصر دنباله.
- **دستورالعمل ۱۰.۵**: درباره لغو کوئری‌های PLINQ

</div>
