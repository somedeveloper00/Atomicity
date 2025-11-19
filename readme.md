<div dir="rtl" style="direction: rtl; unicode-bidi: isolate;">

# کارکرد atomic در ++C در ۵ دقیقه

در این مقاله سعی میکنم در کمترین زمان ممکن، موضوع atomic یا atomicity در ++C رو از سخت‌افزار تا نرم‌افزار و روشهای استاندارد و پیشنهادی رو توضیح بدم.

## در سخت‌افزار چه اتفاقی می‌افته؟

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```cpp
unsigned int takeItem(player& player, item& item)
{
    static unsigned int totalItemsAdded = 0;
    static unsigned int totalSpecialItemsAdded = 0;
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalSpecialItemsAdded++;
    return totalItemsAdded++;
}
```

</div>

دیتای `player` و `items` کجان؟ `this` یا `onItemAdded` کجان؟ هر CPU طبیعتا دیتا رو از DRAM (همون رم عادی) میگیره و میذاره توی cache برای استفاده. cache به قسمتهای کوچکتری به نام cacheline تقسیم میشه. معمولا هر cacheline ۶۴ بایته.  
وقتی یک دیتایی توی cache نیست و باید از DRAM گرفته بشه، بهش میگن cache-miss. هر cache-miss تقریبا ۱۰ برابر یک خوندن از cache طول میکشه. موقع cache-miss شدن، بحای اینکه همه ی cache با هم sync بشه، cacheline ها sync میشه طبق نیاز.  
یعنی در مثال بالا، وقتی `player`خونده میشه، اگر در cache وجود نداشته باشه، یک cacheline انتخاب میشه و  `player` از DRAM خونده میشه و در اون cacheline میشینه. این اتفاق برای `item` و همه دیتای موجود در هر قسمت از برنامه اتفاق می‌افته. به همین دلیله که دیتا هایی که از هم فاصله کمتری دارن، cachemiss کمتری ایجاد میکنند و باعث افزایش سرعت برنامه میشن.


![image01.png](https://github.com/somedeveloper00/Atomicity/blob/main/res/image01.png?raw=true)

## عملهای atomic چه کار میکنند؟
بصورت خلاصه، تضمین میکنن که دیتایی که در cacheline ادیت میشه، بین همه CPU در موقع نیاز، sync میشه. این برای برنامه هایی که از چند thread همزمان استفاده میکنن، قابلیت هایی از جمله mutex و lock-free programming رو ممکن میکنه.

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```diff
unsigned int takeItem(player& player, item& item)
{
    static unsigned int totalItemsAdded = 0;
    static unsigned int totalISpecialItemsAdded = 0;
+   std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalISpecialItemsAdded++;
    return totalItemsAdded++;
}

```
</div>

در مثال بالا، چند thread همزمان میتونن از این تابع استفاده کنن بدون اینکه cacheline هاشون همزمان `items` رو ادیت کنه و باعث اختلال در عمکرد همدیگه بشن. (بدون mutex در مثال بالا، ممکنه دو thread همزمان `items` رو ادیت کنن بدون اینکه تغییرات همدیگه رو ببینن و در نهایت `items` داخل DRAM خراب بشه.)

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```diff
unsigned int takeItem(player& player, item& item)
{
-   static unsigned int totalItemsAdded = 0;
-   static unsigned int totalISpecialItemsAdded = 0;
+   static std::atomic_uint totalItemsAdded{0};
+   static std::atomic_uint totalSpecialItemsAdded{0};
    std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
-       totalISpecialItemsAdded++;
+       totalSpecialItemsAdded.fetch_add(1);
-   return totalItemsAdded++;
+   return totalItemsAdded.fetch_add(1);
}
```
</div>

در این مثال، `totalItemsAdded` به صورت atomic تعریف شده. این یعنی وقتی چند thread همزمان از این تابع استفاده میکنن، مقدار `totalItemsAdded` به درستی بین همه thread ها sync میشه و هیچ thread ای مقدار اشتباهی نمیبینه.

طرز عملکرد atomic ها اینطوریه که طبق استانداردهای تعریف شده و با تعامل با CPU های همسایه، cacheline خودشون رو با هم sync میکنن (cache coherency). یعنی تضمین میکنه که cacheline یی که میخواد تغییر کنه، قبل از تغییر sync شده و هنگام تغییر کردنش هم CPU دیگه ای همون دیتا رو تغییر نمیده. استاندارد معروفی که برای این کار وجود داره، [MESI](https://en.wikipedia.org/wiki/MESI_protocol) هستش که با tag گذاری در cacheline ها باعث تعامل راحت و سریع بین CPU ها میشه.

## استفاده بهتر از cacheline
مموری ای که برای atomic ها استفاده میشه، فرقی با مموری های دیگه نداره. این فقط ++C هستش که برای راحت تر کردن کار، متغییر رو طوری نشون میده که انگار خود متغییر خاص هستش. درواقع صرفا از دستورات مربوط به atomicity در assembly استفاده میشه برای استفاده از اون قسمت مموری. 
این به این معنیه که مثل بقیه مموری ها در ++C، ممکنه بیشتر از یک متغییر در یک cacheline قرار بگیره. در مثال بالا، ممکنه `totalItemsAdded` و `totalSpecialItemsAdded` در یک cacheline قرار بگیرن و تغییر `totalItemsAdded` هر بار باعث sync شدن `totalSpecialItemsAdded` هم بشه. ممکنه نصف `totalItemsAdded` در یک cacheline باشه و نصف دیگش در یک cachline دیگه. برای جلوگیری از این مشکل، بهتره از `alignas` استفاده کنیم تا مطمئن بشیم هر کدوم از متغییر های atomic فقط یک cacheline منحصر بفرد داشته باشن.

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```diff
unsigned int takeItem(player& player, item& item)
{
-   static std::atomic_uint totalItemsAdded{0};
-   static std::atomic_uint totalISpecialItemsAdded{0};
+   static alignas(std::hardware_destructive_interference_size)
+   std::atomic_uint totalItemsAdded{0};
+   static alignas(std::hardware_destructive_interference_size)
+   std::atomic_uint totalSpecialItemsAdded{0};
    std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalSpecialItemsAdded.fetch_add(1);
    return totalItemsAdded.fetch_add(1);
}
```
</div>

## موضوعهای بعدی برای ادامه
با استفاده از atomic ها میشه کارهای جالبی کرد که برنامه های multi-threaded از تمام قدرت CPU هایی که در اختیار ما و استفاده کننده‌های برنامه‌هامون هستش رو استفاده کنن.
در این مقاله سعی کردم به صورت خلاصه و سریع، atomic ها رو توضیح بدم، اما این تازه شروع موضوع atomic هاست و استفاده های پیشرفته‌تر زیای وجود داره. چند مثال میزنم برای اینکه ایده ای داشته باشید برای ادامه دادن و اینکه دنبال چی باشید:
#### CAS (Compare and Swap)
یکی از دستورهای atomic که بیشتر استفاده میشه. به صورت کلی به عمل مقایسه و تعویض گفته میشه. یعنی یک متغییر رو با یک مقدار مقایسه میکنه و اگر برابر بود، مقدار جدیدی رو در اون متغییر قرار میده. این کار باعث میشه که چند thread همزمان بتونن از یک متغییر استفاده کنن بدون اینکه مزاحم هم بشن. این کار در assembly به صورت `cmpxchg` انجام میشه و دست به OS نمیزنه، خودش هم یک دستور atomic هستش، یعنی شامل تمام بهینه‌سازی های atomicity استانداردهایی مثل MESI میشه.
#### MCMP
به تابع ها و سیستمهایی گفته میشه که همزمان چند writer و چند reader را پشتیبانی میکنند. multi-consumer-multi-producer یا برعکس، multi-producer-multi-consumer (MPMC). نوشتن این سیستمها با mutex به راحتی قابل انجامه، اما خب باعث lock های زیادی میشه.
#### lock
اصطلاحیه که برای منتظر موندن یک thread با استفاده از دستورهای سیستم‌عامل استفاده میشه. مثلا مثال قبلی ای که زده بودم، `itemsMutex` وقتی به `std::lock_guard<std::mutex>` میرسه، اون پشتها با استفاده از atomic ها، چک میشه که الآن میشه جلو رفت یا خیر. اگه نشد، معمولا یکم spin-lock میکنن و بعد اگه هنوزم نمیشد پیش رفت، به OS میگن که thread رو بخوابونه. این کار خیلی گرون تموم میشه (بیدار کردن thread در ویندوز حدود ۱۰μs الی ۵۰ μs طول میکشه!) 
#### spin-lock
بعضی وقتها به دلیل گرون بودن lock، ترجیح میدیم که داخل یک حلقه `while` ساده منتظر یک `flag` یی باشیم. بهتره داخل این حلقه از دستورهای مخصوصی استفاده کنیم که به CPU بگه که thread ما داره وقت میکشه و میتونه آسون بگیره (باعث صرفه‌جویی مصرف برق و کنترل حرارت CPU میشه). برای هر سیستم فرق میکنه اما استانداردهایی هم وجود داره، این شروع خوبیه:

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```cpp
#if defined(_MSC_VER) && (defined(_M_IX86) || defined(_M_X64))
#include <immintrin.h>
#define relaxCpu() _mm_pause()
#elif defined(__i386__) || defined(__x86_64__)
#include <xmmintrin.h>
#define relaxCpu() _mm_pause()
#elif defined(__aarch64__) || defined(__arm__)
#define relaxCpu() __asm__ volatile("yield")
#else
#define relaxCpu() std::this_thread::yield()
#endif
```
</div>

#### flag
این رو با کد نشون بدیم راحت تره

<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```cpp
alignas(std::hardware_destructive_interference_size) std::atomic_flag flag{};
void addItem(const item& item)
{
    while (flag.test_and_set(std::memory_order_acquire))
        relaxCpu();
    player.items.push_back(item);
    flag.clear(std::memory_order_release);
}

```
</div>

#### wait
مثل lock ولی spin-lock رو هم شامل میشه. یعنی کلا منتظر موندن یک thread، عملا هدر رفتن CPU.
#### lock-free programming
به برنامه نویسی ای گفته میشه که در اون از mutex و lock استفاده نمیشه. این کار باعث میشه که برنامه نویس بتونه از تمام قدرت CPU استفاده کنه و در عین حال، thread ها همدیگه رو block نکنن. این کار با استفاده از atomic ها انجام میشه و به صورت کلی، یک روش پیچیده تر برای نوشتن برنامه های multi-threaded هستش. نتیجه‌ش همچین چیزی میشه:
<div dir="ltr" style="direction: ltr; unicode-bidi: isolate;">

```cpp
void addItem(const item& item)
{
    player.lockFreeItems.push_back(item);
}

void readItems()
{
    for (const auto& item : player.lockFreeItems)
        std::invoke(onItemAdded, item);
}
```
</div>

به شکلی که هر دوی تابع ها میتونن همزمان اجرا بشن و کد هم ظاهرا و هم باطنن باعث lock شدن thread نمیشه. یکی از کتابخونه های معروف که دیتای lock-free داره، boost هستش. البته باعث نمیشه wait نداشته باشه!
#### wait-free programming
مثل lock-free ولی حتی wait هم نداره! این سختترین قسمت استفاده از atomic هاست و نتیجه‌ش میشه سیستمی که از تمام قدرت چند CPU همزمان استفاده میکنه و هیچ cycleیی هدر نمیره.

</div>

