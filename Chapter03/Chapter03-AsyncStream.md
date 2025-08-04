

<div dir="rtl" align="right">

# فصل سوم: جریان‌های ناهمگام (Asynchronous Streams)

جریان‌های ناهمگام راهی هستند برای دریافت چندین داده به‌صورت ناهمگام. این جریان‌ها مبتنی بر enumerableهای ناهمگام یا همان `IAsyncEnumerable<T>` ساخته شده‌اند. یک enumerable ناهمگام، نسخهٔ ناهمگامی از یک enumerable معمولی است؛ یعنی می‌تواند آیتم‌ها را در صورت نیاز برای مصرف‌کننده فراهم کند و هر آیتم می‌تواند به صورت ناهمگام تولید شود.

من متوجه شده‌ام که مقایسه جریان‌های ناهمگام با انواع دیگری که ممکن است آشناتر باشند و بررسی تفاوت‌هایشان، بسیار مفید است. این کار کمک می‌کند تا بهتر به خاطر بسپارم چه زمانی باید از جریان‌های ناهمگام استفاده کنم و چه زمانی انواع دیگر مناسب‌تر هستند.

---

## جریان‌های ناهمگام و `Task<T>`

روش استاندارد ناهمگام با استفاده از `Task<T>` فقط برای مدیریت ناهمگام یک مقدار داده مناسب است. وقتی یک `Task<T>` به پایان می‌رسد، دیگر تمام است؛ یک `Task<T>` نمی‌تواند بیش از یک مقدار از نوع `T` را برای مصرف‌کننده‌هایش فراهم کند. حتی اگر `T` یک مجموعه باشد، فقط یک بار می‌تواند آن مقدار را ارائه دهد.

برای اطلاعات بیشتر درباره استفاده از `async` با `Task<T>`, به بخش "مقدمه‌ای بر برنامه‌نویسی ناهمگام" و فصل دوم مراجعه کنید.

وقتی `Task<T>` را با جریان‌های ناهمگام مقایسه می‌کنیم، جریان‌های ناهمگام بیشتر شبیه به enumerableها هستند. به طور خاص، یک `IAsyncEnumerator<T>` می‌تواند هر تعداد مقدار از نوع `T` را، یکی‌یکی، فراهم کند. درست همانند `IEnumerator<T>`, یک `IAsyncEnumerator<T>` حتی می‌تواند از لحاظ طولی بی‌نهایت باشد.

---

## جریان‌های ناهمگام و `IEnumerable<T>`

همان‌طور که از نامش پیداست، `IAsyncEnumerable<T>` مشابه `IEnumerable<T>` است. احتمالاً این نکته تعجب‌آور نیست؛ هر دو به مصرف‌کنندگان امکان می‌دهند که عناصر را به صورت یکی‌یکی از آن‌ها دریافت کنند. تفاوت اصلی در نامشان است: یکی ناهمگام است و دیگری نه.

وقتی کد شما روی یک `IEnumerable<T>` پیمایش می‌کند (iterate)، هنگام دریافت هر عنصر، عملیات را *مسدود* (block) می‌کند. اگر `IEnumerable<T>` در واقع نشان‌دهنده عملیاتی مبتنی بر I/O مانند پرس‌وجوی پایگاه داده یا فراخوانی یک API باشد، کد مصرف‌کننده در نهایت روی عملیات I/O مسدود می‌شود که ایده‌آل نیست.

`IAsyncEnumerable<T>` دقیقاً شبیه به یک `IEnumerable<T>` کار می‌کند، با این تفاوت که هر عنصر بعدی را به‌صورت ناهمگام بازیابی می‌کند.

---

## جریان‌های ناهمگام و `Task<IEnumerable<T>>`

کاملاً ممکن است که به صورت ناهمگام یک مجموعه با بیش از یک آیتم را بازگردانیم؛ یک مثال رایج، `Task<List<T>>` است. با این حال، متدهای async که یک `List<T>` بازمی‌گردانند، فقط یک دستور بازگشت (`return`) دارند؛ مجموعه باید قبل از بازگردانده شدن، کاملاً پر شده باشد.

حتی متدهایی که `Task<IEnumerable<T>>` بازمی‌گردانند ممکن است یک enumerable را به صورت ناهمگام برگردانند، اما آن enumerable به صورت همگام ارزیابی می‌شود. برای مثال، LINQ-to-Entities دارای متد LINQ به نام `ToListAsync` است که یک `Task<List<T>>` بازمی‌گرداند. وقتی یک provider پایگاه داده LINQ این دستور را اجرا می‌کند، اول باید با پایگاه داده ارتباط برقرار کند و همه پاسخ‌های مطابقت‌یافته را دریافت کند تا لیست را به طور کامل پر کرده و بازگرداند.

### محدودیت‌های `Task<IEnumerable<T>>`

محدودیت `Task<IEnumerable<T>>` این است که نمی‌تواند آیتم‌ها را به‌محض دریافت شدن بازگرداند؛ اگر مجموعه‌ای بازمی‌گردد، باید همه عناصر را در حافظه بارگذاری کند، مجموعه را پر کرده و سپس کل مجموعه را به یک‌باره بازگرداند. حتی اگر یک پرس‌وجوی LINQ را بازگرداند، می‌تواند آن پرس‌وجو را به‌صورت ناهمگام بسازد، اما پس از بازگشت پرس‌وجو، هر آیتم به صورت همگام از آن استخراج می‌شود.

---

## چرا `IAsyncEnumerable<T>`؟

`IAsyncEnumerable<T>` هم چندین آیتم را به صورت ناهمگام بازمی‌گرداند، اما تفاوت اینجاست که `IAsyncEnumerable<T>` می‌تواند برای هر آیتم به صورت ناهمگام عمل کند. این یک جریان واقعاً ناهمگام از آیتم‌هاست.

</div>

---

### نمونه کد (چپ‌چین)

```csharp
await foreach (var item in GetItemsAsync())
{
    Console.WriteLine(item);
}
// فرض تابع زیر یک جریان ناهمگام برمی‌گرداند:
public async IAsyncEnumerable<int> GetItemsAsync()
{
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(1000);
        yield return i;
    }
}
```

---



<div dir="rtl" align="right">

# جریان‌های ناهمگام و `IObservable<T>`

Observableها، نمونهٔ واقعی جریان‌های ناهمگام‌اند؛ اعلان‌های خود را یکی‌یکی و با پشتیبانی واقعی از تولید ناهمگام (بدون مسدود شدن یا *blocking*) ارسال می‌کنند. اما الگوی مصرف (**Consumption Pattern**) برای `IObservable<T>` کاملاً متفاوت از `IAsyncEnumerable<T>` است.  
برای اطلاعات بیشتر درباره‌ی `IObservable<T>` به فصل ششم مراجعه کنید.

برای مصرف یک `IObservable<T>`, کد باید یک کوئری شبیه LINQ تعریف کند که اعلان‌های Observable از طریق آن جریان پیدا کند و سپس برای شروع جریان داده‌ها، بر روی Observable مشترک شود (*subscribe*). هنگام کار با Observableها، کد ابتدا تعریف می‌کند که چگونه به اعلان‌های ورودی واکنش نشان دهد و سپس آن‌ها را فعال می‌کند (به همین دلیل به آن “reactive” می‌گویند).  
در مقابل، مصرف کردن یک `IAsyncEnumerable<T>` بسیار شبیه به مصرف `IEnumerable<T>` است، با این تفاوت که مصرف به صورت ناهمگام انجام می‌شود.

همچنین مشکلی به نام **فشار برگشتی (backpressure)** وجود دارد؛ تمام اعلان‌ها در System.Reactive به صورت همگام (*synchronous*) ارسال می‌شوند. بنابراین به‌محض اینکه یک اعلان برای یک مقدار به مشترکین ارسال شود، Observable بلافاصله ادامه داده و مقدار بعدی را برای نشر دریافت می‌کند و ممکن است دوباره API را فراخوانی کند. اگر کد مصرف‌کننده در حال مصرف جریان به صورت ناهمگام باشد (یعنی برای هر اعلان، یک اقدام ناهمگام انجام دهد)، Observable از کد مصرف‌کننده جلو می‌زند.

یک راه خوب برای فکر کردن به تفاوت بین آن‌ها این است که `IObservable<T>` مبتنی بر **push** است و `IAsyncEnumerable<T>` مبتنی بر **pull**.  
یک جریان observable اعلان‌ها را به کد شما فشار می‌دهد (push)، اما یک جریان ناهمگام اجازه می‌دهد که کد شما داده‌ها را به صورت (ناهمگام) از آن بیرون بکشد (pull). تنها زمانی که کد مصرف‌کننده آیتم بعدی را درخواست کند، جریان observable اجرای خود را ادامه می‌دهد.

---

## خلاصه تفاوت‌ها (مثال نظری)

خیلی از APIها پارامترهای `offset` و `limit` دارند تا امکان صفحه‌بندی نتایج را فراهم کنند. فرض کنید متدی می‌خواهیم بنویسیم که نتایج را از یک API با قابلیت صفحه‌بندی دریافت و مدیریت کند:

 اگر متد ما `Task<T>` بازگرداند:  
فقط یک مقدار بازمی‌گردانیم. مناسب برای یک بار فراخوانی API.

 اگر متد ما `IEnumerable<T>` بازگرداند:  
حلقه زده و نتایج صفحه‌بندی را دریافت می‌کنیم. اما نمی‌توان این کار را **ناهمگام** انجام داد، فراخوانی‌های API باید همگام باشند.

 اگر متد ما `Task<List<T>>` بازگرداند:  
می‌توانیم نتایج را **ناهمگام** دریافت کنیم، اما باید همه را جمع کنیم و در انتها همه را یکجا بازگردانیم، نه هر آیتم را به‌محض دریافت.

 اگر متد ما `IObservable<T>` بازگرداند:  
با System.Reactive یک Observable واقعی می‌سازیم که با Subscription شروع می‌شود و هر آیتم را به محض دریافت انتشار می‌دهد (push). این مدل برای سناریوهایی مثل دریافت پیام‌هایWebSocket یا SignalR مناسب‌تر است.

 اگر متد ما `IAsyncEnumerable<T>` بازگرداند:  یک حلقه await/yield return داریم و جریان ناهمگام به صورت طبیعی و pull ساخته می‌شود؛ برای مدیریت صفحه‌بندی و دریافت داده‌های واکنشیِ قابل کنترل بسیار مناسب است.

---

## جدول ۳-۱: دسته‌بندی انواع متداول جریان‌ها

| نوع                | تک یا چند مقداری | همگام یا ناهمگام | مبتنی بر Push/Pull  |
|--------------------|:----------------:|:----------------:|:-------------------:|
| `T`                | تک مقدار         | همگام            | —                   |
| `IEnumerable<T>`   | چند مقدار        | همگام            | —                   |
| `Task<T>`          | تک مقدار         | ناهمگام          | Pull                |
| `IAsyncEnumerable<T>` | چند مقدار     | ناهمگام          | Pull                |
| `IObservable<T>`   | تک یا چند مقدار  | ناهمگام          | Push                |

---

> ⚠ **هشدار:**  
زمان چاپ این کتاب، .NET Core 3.0 هنوز در حالت بتا بود؛ بنابراین جزئیات مربوط به جریان‌های ناهمگام ممکن است تغییر کند.

---

# ۳.۱ ساخت جریان‌های ناهمگام

## مسئله

نیاز دارید چندین مقدار را بازگردانید و هر مقدار ممکن است به عملی ناهمگام نیاز داشته باشد. معمولاً یکی از این سناریوهاست:
- چند مقدار دارید (مثل `IEnumerable<T>`) و نیاز دارید کار ناهمگامی روی آن انجام شود.
- یک مقدار ناهمگام دارید (مثل `Task<T>`) و بعدها نیاز است چند مقدار دیگر بازگردد.

---

## راه‌حل

ترکیب `yield return` (برای چند مقدار) با `async/await` (برای ناهمگامی). کافی است نوع خروجی متد را `IAsyncEnumerable<T>` تعریف کنید:

### نمونه کد ساده (چپ‌چین)

<div dir="ltr" align="left">

```csharp
async IAsyncEnumerable<int> GetValuesAsync()
{
    await Task.Delay(1000); // انجام کار ناهمگام
    yield return 10;
    await Task.Delay(1000); // انجام کار ناهمگام دیگر
    yield return 13;
}
```

</div>

این مثال نشان می‌دهد چگونه از `await` در کنار `yield return` برای ساخت جریان ناهمگام استفاده کنیم.

---

### مثال واقعی‌تر: صفحه‌بندی نتایج API

<div dir="ltr" align="left">

```csharp
async IAsyncEnumerable<string> GetValuesAsync(HttpClient client)
{
    int offset = 0;
    const int limit = 10;
    while (true)
    {
        // دریافت صفحه کنونی
        string result = await client.GetStringAsync(
           $"https://example.com/api/values?offset={offset}&limit={limit}");

        string[] valuesOnThisPage = result.Split('\n');

        foreach (string value in valuesOnThisPage)
            yield return value;

        if (valuesOnThisPage.Length != limit)
            break;

        offset += limit;
    }
}
```

</div>

زمانی که این متد اجرا شود، ابتدا یک صفحه را ناهمگام دریافت کرده، هر عنصر آن را یکی‌یکی بازمی‌گرداند، و تنها اگر نیاز باشد صفحه بعدی را دریافت می‌کند.

---

## بحث

از زمان معرفی `async` و `await` همیشه این پرسش مطرح بود که چگونه آن‌ها را با `yield return` ترکیب کنیم. سال‌ها این کار ممکن نبود؛ اما اکنون جریان‌های ناهمگام آن را به C# و نسخه‌های مدرن دات‌نت آورده‌اند.

در مثال بالا می‌بینید که معمولاً فقط بعضی آیتم‌ها نیاز به انجام کار ناهمگام دارند. مثلاً اگر سایز صفحه ۱۰ باشد، فقط حدود ۱ از هر ۱۰ عنصر نیازمند عملیات ناهمگام است.

جریان‌های ناهمگام بهینه‌اند، چرا که همگام و ناهمگام بودن را ترکیب می‌کنند. به همین دلیل بر پایه `ValueTask<T>` ساخته شده‌اند؛ هرگاه آیتم‌ها را همگام یا ناهمگام دریافت کنید، سریع و کم‌مصرف خواهند بود.

### توجه به لغو (Cancellation)

در پیاده‌سازی این جریان‌ها، به پشتیبانی از لغو (CancellationToken) نیز توجه کنید. برخی سناریوها نیاز به لغو واقعی دارند (مانند زمانی که کاربر دکمه لغو را می‌زند)، برخی نه.  
برای جزییات بیشتر، به دستورپخت‌های ۳.۲ (مصرف جریان‌های ناهمگام)، ۳.۴ (مدیریت لغو)، و ۲.۱۰ (ValueTask<T>) رجوع کنید.

---

## همچنین ببینید:
- **دستور پخت ۳.۲:** مصرف جریان‌های ناهمگام  
- **دستور پخت ۳.۴:** مدیریت لغو در جریان‌های ناهمگام  
- **دستور پخت ۲.۱۰:** توضیح بیشتری درباره ValueTask<T> و زمان استفاده آن  

</div>

<div dir="rtl" align="right">


# ۳.۲ مصرف جریان‌های ناهمگام

## مسئله

شما باید نتایج یک **جریان ناهمگام** (*asynchronous stream*) یا همان *enumerable* ناهمگام را پردازش کنید.

---

## راه‌حل

مصرف یک عملیات ناهمگام با `await` انجام می‌شود و در enumerable معمولی از `foreach` استفاده می‌کنیم. برای مصرف یک enumerable ناهمگام، این دو را ترکیب کرده و از `await foreach` استفاده می‌کنیم.

### نمونه ساده:

فرض کنید یک جریان ناهمگام داریم که روی پاسخ‌های یک API پیمایش می‌کند:

<div dir="ltr" align="left">


```csharp
IAsyncEnumerable<string> GetValuesAsync(HttpClient client);

public async Task ProcessValueAsync(HttpClient client)
{
    await foreach (string value in GetValuesAsync(client))
    {
        Console.WriteLine(value);
    }
}
```

</div>

#### چه اتفاقی رخ می‌دهد؟
- متد `GetValuesAsync` فراخوانی شده و یک `IAsyncEnumerable<T>` بازمی‌گرداند.
- حلقه foreach یک enumerator ناهمگام می‌سازد.
- عملیات «دریافت عنصر بعدی» ممکن است *ناهمگام* باشد، پس تا زمانی که عنصر بعدی برسد یا جریان تمام شود، منتظر می‌ماند (`await`).
- اگر عنصری رسید، حلقه اجرا می‌شود؛ اگر جریان تمام شود، حلقه پایان می‌یابد.

---

### پردازش ناهمگام هر عنصر

بدنه حلقه می‌تواند صرفاً همگام باشد یا برای هر عنصر عملی ناهمگام انجام دهد:

<div dir="ltr" align="left">


```csharp
IAsyncEnumerable<string> GetValuesAsync(HttpClient client);

public async Task ProcessValueAsync(HttpClient client)
{
    await foreach (string value in GetValuesAsync(client))
    {
        await Task.Delay(100); // کار ناهمگام روی هر آیتم
        Console.WriteLine(value);
    }
}
```
</div>

در این حالت، حلقه تا پایان پردازش عنصر فعلی منتظر می‌ماند و سپس سراغ عنصر بعدی می‌رود.

---

### استفاده از ConfigureAwait

در حلقه `await foreach`، یک `await` مخفی وجود دارد. می‌توانید با `ConfigureAwait(false)` اجرای حلقه و عملیات را از context فعلی جدا کنید.

<div dir="ltr" align="left">


```csharp
IAsyncEnumerable<string> GetValuesAsync(HttpClient client);

public async Task ProcessValueAsync(HttpClient client)
{
    await foreach (string value in GetValuesAsync(client).ConfigureAwait(false))
    {
        await Task.Delay(100).ConfigureAwait(false); // کار ناهمگام
        Console.WriteLine(value);
    }
}
```

</div>

---

## بحث و نکات تکمیلی

- `await foreach` طبیعی‌ترین و راحت‌ترین روش مصرف جریان‌های ناهمگام است.
- پشتیبانی از `ConfigureAwait(false)`، برای جلوگیری از نگه داشتن context، در حلقه‌های `await foreach` وجود دارد.
- امکان پاس دادن توکن لغو (`CancellationToken`) نیز در این حلقه‌ها پشتیبانی می‌شود که موضوع دستور پخت ۳.۴ است.
- بدنه حلقه می‌تواند همگام یا ناهمگام باشد.
- در مقایسه با انتزاع‌هایی مانند `IObservable<T>`, مصرف کاملاً ناهمگام یک جریان در C# با `await foreach` فوق‌العاده ساده و کارا انجام می‌شود.
- حلقه `await foreach` می‌تواند هم یک `await` برای «دریافت عنصر بعدی» داشته باشد و هم، اگر نیاز باشد (مثلاً در disposable), برای پاک‌سازی ناهمگام enumerable هم یک `await` دیگر تولید کند.

---

## همچنین ببینید

- **دستور پخت ۳.۱:** تولید جریان‌های ناهمگام  
- **دستور پخت ۳.۳:** متدهای LINQ متداول برای جریان‌های ناهمگام  
- **دستور پخت ۳.۴:** مدیریت لغو در جریان‌های ناهمگام  
- **دستور پخت ۱۱.۶:** پاک‌سازی ناهمگام (*Asynchronous Disposal*)

</div>





<div dir="rtl" align="right">

# ۳.۳ استفاده از LINQ با جریان‌های ناهمگام

## مسئله

می‌خواهید یک **جریان ناهمگام** (asynchronous stream) را با عملگرهای شناخته‌شده و تست‌شده LINQ پردازش کنید.

---

## راه‌حل

- `IEnumerable<T>` دارای LINQ to Objects است؛  
- `IObservable<T>` نیز LINQ to Events را دارد.
- برای `IAsyncEnumerable<T>`، کتابخانه‌ای تحت NuGet با نام **System.Linq.Async** به شما LINQ برای جریان‌های ناهمگام می‌دهد.

### فیلتر ناهمگام: WhereAwait

برای توابع شرطی ناهمگام (مثلاً چک کردن هر ایتم روی API):

<div dir="ltr" align="left">


```csharp
IAsyncEnumerable<int> values = SlowRange().WhereAwait(
    async value =>
    {
        await Task.Delay(10); // کار ناهمگام برای تصمیم
        return value % 2 == 0;
    });

await foreach (int result in values)
{
    Console.WriteLine(result);
}

// تابع تولید اعداد به صورت ناهمگام و کندشونده
async IAsyncEnumerable<int> SlowRange()
{
    for (int i = 0; i != 10; ++i)
    {
        await Task.Delay(i * 100);
        yield return i;
    }
}

```

</div>

### فیلتر همگام استاندارد

استفاده از عملگرهای استاندارد نیز ممکن است؛ خروجی همچنان ناهمگام است:

<div dir="ltr" align="left">


```csharp
IAsyncEnumerable<int> values = SlowRange().Where(
    value => value % 2 == 0);

await foreach (int result in values)
{
    Console.WriteLine(result);
}
```
</div>

### همه عملگرها در دسترس‌اند

عملگرهایی مثل `Where`، `Select`، `SelectMany`، حتی `Join` و...  
اکثر این عملگرها در نسخه ناهمگام خود (`Await`) نیز برای delegateهای ناهمگام موجودند.

#### پسوندهای Await و Async در نام متدها

- پسوند **Await** برای متدهایی است که delegate ناهمگام می‌گیرند و با await کار می‌کنند (`WhereAwait`, `SelectAwait`)
- پسوند **Async** فقط برای عملگرهای پایانی است که مقدار نهایی می‌سازند (`CountAsync`)
- اگر هر دو را دارد، هم شرط ناهمگام می‌گیرد هم مقدار نهایی بازمی‌گرداند (`CountAwaitAsync`)

##### نمونه عملگر پایانی:

<div dir="ltr" align="left">


```csharp
int count = await SlowRange().CountAsync(
    value => value % 2 == 0);

int count = await SlowRange().CountAwaitAsync(
    async value =>
    {
        await Task.Delay(10);
        return value % 2 == 0;
    });
```

</div>

### نکته نام‌گذاری

- عملگرهایی که مقدار نهایی برمی‌گردانند —> آخرشان Async
- عملگرهایی که delegate ناهمگام می‌گیرند —> Await دارند
- عملگرهایی که هر دو را انجام می‌دهند —> AwaitAsync دارند

> **کتابخانه System.Linq.Async در NuGet**: همه این متدها را فراهم می‌کند.  
> کتابخانه‌ی اضافی: System.Interactive.Async  

---

## بحث

- جریان‌های ناهمگام Pull-based هستند؛ عملگرهایی مانند Throttle یا Sample (که در Observableها وجود دارد) اینجا معنی ندارند.
- می‌توانید هر `IEnumerable<T>` را با `ToAsyncEnumerable()` به جریان ناهمگام تبدیل کنید و از عملگرهای Await استفاده کنید.

---

### همچنین ببینید  
- **دستور پخت ۳.۱:** تولید جریان‌های ناهمگام  
- **دستور پخت ۳.۲:** مصرف جریان‌های ناهمگام  

---

# ۳.۴ جریان‌های ناهمگام و لغو (Cancellation)

## مسئله

بعضی وقت‌ها نیاز دارید جریان ناهمگام را لغو کنید (استفاده از CancellationToken).

---

## راه‌حل

۱. **لغو با break:**  
   اگر کافیست فقط هنگام رسیدن به شرطی خاص حلقه را قطع کنید، نیازی به لغو واقعی نیست:

<div dir="ltr" align="left">


```csharp
await foreach (int result in SlowRange())
{
    Console.WriteLine(result);
    if (result >= 8)
        break;
}

async IAsyncEnumerable<int> SlowRange()
{
    for (int i = 0; i != 10; ++i)
    {
        await Task.Delay(i * 100);
        yield return i;
    }
}
```

</div>

۲. **لغو واقعی با CancellationToken:**  
   وقتی بخواهید از بیرون حلقه را متوقف کنید، باید متد جریان ناهمگام شما یک پارامتر CancellationToken بگیرد (و با ویژگی [EnumeratorCancellation] مشخص شود):

<div dir="ltr" align="left">


```csharp
using var cts = new CancellationTokenSource(500); // لغو پس از ۵۰۰میلی‌ثانیه
CancellationToken token = cts.Token;
await foreach (int result in SlowRange(token))
{
    Console.WriteLine(result);
}

async IAsyncEnumerable<int> SlowRange(
    [EnumeratorCancellation] CancellationToken token = default)
{
    for (int i = 0; i != 10; ++i)
    {
        await Task.Delay(i * 100, token);
        yield return i;
    }
}
```
  
</div>

۳. **لغو برای هر enumeration مجزا:**
   می‌توانید توکن لغوی برای هر enumerator مجزا به کمک WithCancellation پاس دهید:

<div dir="ltr" align="left">


```csharp
async Task ConsumeSequence(IAsyncEnumerable<int> items)
{
    using var cts = new CancellationTokenSource(500);
    CancellationToken token = cts.Token;
    await foreach (int result in items.WithCancellation(token))
    {
        Console.WriteLine(result);
    }
}

async IAsyncEnumerable<int> SlowRange(
    [EnumeratorCancellation] CancellationToken token = default)
{
    for (int i = 0; i != 10; ++i)
    {
        await Task.Delay(i * 100, token);
        yield return i;
    }
}

await ConsumeSequence(SlowRange());
```

</div>

- ویژگی `[EnumeratorCancellation]` باعث می‌شود کامپایلر توکن از WithCancellation را پاس بدهد.
- لغو باعث برگرداندن OperationCanceledException هنگام رسیدن توکن می‌شود.

۴. **همراه با ConfigureAwait:**  
   می‌توانید متدهای WithCancellation و ConfigureAwait(false) را زنجیره‌ای استفاده کنید:

<div dir="ltr" align="left">


```csharp
async Task ConsumeSequence(IAsyncEnumerable<int> items)
{
    using var cts = new CancellationTokenSource(500);
    CancellationToken token = cts.Token;
    await foreach (int result in items.WithCancellation(token).ConfigureAwait(false))
    {
        Console.WriteLine(result);
    }
}
```

</div>

---

## بحث  

 لغو ویژه‌ی enumerator است نه enumerable؛ هر enumeration جدید می‌تواند توکن متفاوت بگیرد.
 
 WithCancellation، ساده‌ترین راه لغو امن و شفاف برای هر حلقه است.

---

### همچنین ببینید  
- **دستور پخت ۳.۱:** تولید جریان‌های ناهمگام  
- **دستور پخت ۳.۲:** مصرف جریان‌های ناهمگام  
- **فصل ۱۰:** لغو مشارکتی (Cooperative Cancellation)  

</div>
