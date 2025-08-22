
# 📚 ترجمه کتاب Concurrency in C# Cookbook - ویرایش دوم

> **برنامه‌نویسی ناهمزمان، موازی و چندنخی در C#**
> 
> نویسنده: **Stephen Cleary**  

<div dir="rtl" align="right">


## 📋 درباره کتاب

این کتاب یکی از معتبرترین منابع یادگیری برنامه‌نویسی همزمان (Concurrency) در C# محسوب می‌شود. این اثر به صورت Cookbook (دستورالعمل) نوشته شده و روش‌های عملی و کاربردی برای پیاده‌سازی الگوهای مختلف برنامه‌نویسی ناهمزمان، موازی و چندنخی ارائه می‌دهد.


</div>

<div dir="rtl" align="right">


### 🌟 نظرات کارشناسان

> **"چالش بعدی بزرگ در محاسبات، در دسترس قرار دادن موازی‌سازی عظیم برای افراد عادی است. توسعه‌دهندگان قدرت بیشتری نسبت به گذشته در اختیار دارند، اما بیان همزمانی هنوز هم برای بسیاری چالش‌برانگیز است. استیون توجه خود را به این مشکل معطوف کرده و به همه ما کمک می‌کند تا همزمانی، threading، مدل‌های برنامه‌نویسی واکنشی، موازی‌سازی و خیلی موارد دیگر را بهتر درک کنیم."**
> 
> — **Scott Hanselman**, مدیر ارشد برنامه در Microsoft

> **"گستردگی تکنیک‌های پوشش داده شده و فرمت cookbook این کتاب را به مرجع ایده‌آلی برای همزمانی مدرن .NET تبدیل می‌کند."**
> 
> — **Jon Skeet**, مهندس ارشد نرم‌افزار در Google

</div>

<div dir="rtl" align="right">


## 📑 فهرست مطالب

| فصل | عنوان | توضیحات |
|------|--------|----------|
| **مقدمه** | [Book Translate](./Intro.md) | معرفی کتاب و مفاهیم اولیه |
| **فصل ۱** | [مروری کلی](./Chapter01/Chapter01-overview.md) | آشنایی با مفاهیم پایه همزمانی |
| **فصل ۲** | [مبانی Async](./Chapter02/Chapter02-AsyncBasic.md) | کار با async/await و Task |
| **فصل ۳** | [جریان‌های ناهمزمان](./Chapter03/Chapter03-AsyncStream.md) | IAsyncEnumerable و AsyncStreams |
| **فصل ۴** | [مبانی موازی‌سازی](./Chapter04/Chapter04-ParallelBasics.md) | Parallel.ForEach و PLINQ |
| **فصل ۵** | [مبانی Dataflow](./Chapter05/Chapter05-DataflowBasics.md) | TPL Dataflow و pipeline processing |
| **فصل ۶** | [System.Reactive](./Chapter06/Chapter6_SystemReactive.md) | برنامه‌نویسی واکنشی با Rx.NET |
| **فصل ۷** | [تست](./Chapter07/Chapter7-Testing.md) | تست کد ناهمزمان و موازی |
| **فصل ۸** | [تعامل](./Chapter08/Chapter08-Interop.md) | تعامل با کدهای قدیمی و external |
| **فصل ۹** | [مجموعه‌ها](./Chapter09/Chapter09-Collections.md) | Collections همزمان و thread-safe |
| **فصل ۱۰** | [لغو عملیات](./Chapter10/Chapter10-Cancellation.md) | CancellationToken و لغو عملیات |
| **فصل ۱۱** | [OOP دوستدار تابعی](./Chapter11/Chapter11-FunctionalFriendlyOOP.md) | الگوهای functional در C# |
| **فصل ۱۲** | [همگام‌سازی](./Chapter12/Chapter12-Synchronization.md) | قفل‌ها و همگام‌سازی threads |
| **فصل ۱۳** | [زمان‌بندی](./Chapter13/Chapter13-Scheduling.md) | TaskScheduler و کنترل اجرا |
| **فصل ۱۴** | [سناریوهای عملی](./Chapter14/Chapter14-Scenarios.md) | مثال‌های کاربردی و پیاده‌سازی |

</div>

<div dir="rtl" align="right">


## 🎯 اهداف یادگیری

پس از مطالعه این کتاب، توانایی‌های زیر را خواهید داشت:

- ✅ درک عمیق مفاهیم **async/await** و **Task-based Asynchronous Pattern**
- ✅ پیاده‌سازی **موازی‌سازی** با PLINQ و Parallel class
- ✅ کار با **Reactive Extensions** برای برنامه‌نویسی event-driven
- ✅ استفاده از **TPL Dataflow** برای پردازش pipeline
- ✅ مدیریت **CancellationToken** و لغو عملیات
- ✅ تست کردن کد ناهمزمان و موازی
- ✅ بهینه‌سازی عملکرد اپلیکیشن‌های concurrent

</div>

<div dir="rtl" align="right">


## 🛠️ پیش‌نیازها

- آشنایی با زبان **C#** و **.NET Framework/Core**
- درک مفاهیم پایه **Object-Oriented Programming**
- آشنایی با **Visual Studio** یا **VS Code**

</div>

<div dir="rtl" align="right">

## 🤝 مشارکت در پروژه

مشارکت شما در بهبود این ترجمه بسیار ارزشمند است! برای مشارکت:

. **Fork** کنید

. **Branch** جدید بسازید (`git checkout -b feature/improvement`)

. تغییرات خود را **Commit** کنید (`git commit -am 'بهبود ترجمه فصل X'`)

. **Push** کنید (`git push origin feature/improvement`)

. **Pull Request** ایجاد کنید

</div>

<div dir="rtl" align="right">


### 🎯 راهنمای مشارکت

- از **اصطلاحات فنی فارسی** استفاده کنید
- **قالب‌بندی** یکسان را رعایت کنید
- **مثال‌های کد** را با توضیحات فارسی همراه کنید
- **منابع مفید** اضافی اضافه کنید

</div>

<div dir="rtl" align="right">


## 📄 اطلاعات کپی‌رایت

**عنوان اصلی:** Concurrency in C# Cookbook, Second Edition  
**نویسنده:** Stephen Cleary  
**ناشر:** O'Reilly Media, Inc.  
**سال انتشار:** ۲۰۱۹  
**ISBN:** 978-1-492-05450-4  

</div>

<div dir="rtl" align="right">


### ⚖️ لایسنس

این ترجمه صرفاً برای اهداف **آموزشی** و **غیرتجاری** انجام شده است. تمام حقوق کتاب اصلی متعلق به **Stephen Cleary** و **O'Reilly Media** می‌باشد.

```
Copyright © 2019 Stephen Cleary. All rights reserved.
Published by O'Reilly Media, Inc.
```

</div>

<div dir="rtl" align="right">


## 🔗 منابع مفید

- [مخزن رسمی O'Reilly](http://oreilly.com)
- [وب‌سایت Stephen Cleary](https://blog.stephencleary.com/)
- [اسناد رسمی Microsoft - Async Programming](https://docs.microsoft.com/en-us/dotnet/csharp/async)
 [TPL Dataflow](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/dataflow-task-parallel-library)
 [Reactive Extensions](https://github.com/dotnet/reactive)

</div>

<div dir="rtl" align="right">


## 📊 وضعیت ترجمه

| بخش | وضعیت | درصد تکمیل |
|------|--------|-------------|
| مقدمه | ✅ تکمیل | 100% |
| فصل ۱-۷ | ✅ تکمیل | 100% |
| فصل ۸-۱۴ | ✅ تکمیل | 100% |

</div>

<div dir="rtl" align="right">



## 📞 تماس با من

برای پیشنهادات، انتقادات یا سوالات:

 **GitHub:** [@پروفایل گیت هاب](https://github.com/alirezark70)
- **ایمیل:** rezaee.alireza7098@gmail.com
- **لینکدین:** [پروفایل لینکدین](https://www.linkedin.com/in/alireza-rezaee-developer)
- **تلگرام:** Alirezark70
---

**⭐ اگر این ترجمه برای شما مفید بود، لطفاً ستاره بدهید!**

> "بهترین راه یادگیری، تدریس است" - ریچارد فاینمن
