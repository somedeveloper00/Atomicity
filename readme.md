<div dir="rtl">

# کارکرد atomic در ++C در ۵ دقیقه

در این مقاله سعی میکنم در کمترین زمان ممکن، موضوع atomic یا atomicity در ++C رو از سخت‌افزار تا نرم‌افزار و روشهای استاندارد و پیشنهادی رو توضیح بدم.

## در سخت‌افزار چه اتفاقی میوفته؟

<div dir="ltr">

```cpp
unsigned int takeItem(player& player, item& item)
{
    static unsigned int totalItemsAdded = 0;
    static unsigned int totalISpecialtemsAdded = 0;
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalISpecialtemsAdded++;
    return totalItemsAdded++;
}
```

</div>

پوینتر `player` و `items` کجان؟ پوینتر `this` یا `onItemadded` کجان؟ هر CPU طبیعتا دیتا رو از DRAM (همون رم عادی) میگیره و میذاره توی cache برای استفاده. توی cache، هر یک WORD میشه یک cacheline، که مثلا روی سیستمهای ۶۴ بیتی میشه ۶۴ بیت.  
وقتی یک دیتایی توی cache نیست و باید از DRAM گرفته بشه، بهش میگن cache-miss. هر cache-miss تقریبا ۱۰ برابر یک خوندن از cache طول میکشه. موقع cache-miss شدن، بحای اینکه همه ی cache با هم sync بشه، cacheline ها sync میشه طبق نیاز.  
یعنی در مثال بالا، وقتی `player`خونده میشه، اگر در cache وجود نداشته باشه، یک cacheline انتحاب میشه و پوینتر `player` از DRAM خونده میشه و در اون cacheline میشینه. این اتفاق برای `item` و همه دیتای موجود در هر قسمت از برنامه اتفاق میوفته. به همین دلیله که دیتا هایی که از هم فاصله کمتری دارن، cachemiss کمتری ایجاد میکنند و باعث افزایش سرعت برنامه میشن.


![image01.png](https://github.com/somedeveloper00/Atomicity/blob/main/res/image01.png?raw=true)

## عملهای atomic چه کار میکنند؟
بصورت خلاصه، تضمین میکنن که دیتایی که در cacheline ادیت میشه، بین همه CPU در موقع نیاز، sync میشه. این برای برنامه هایی که از چند thread همزمان استفاده میکنن، قابلیت هایی از جمله mutex و lock-free programming رو ممکن میکنه.

<div dir="ltr">

```diff
unsigned int takeItem(player& player, item& item)
{
    static unsigned int totalItemsAdded = 0;
    static unsigned int totalISpecialtemsAdded = 0;
+   std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalISpecialtemsAdded++;
    return totalItemsAdded++;
}

```
</div>

در مثال بالا، چند thread همزمان میتونن از این تابع استفاده کنن بدون اینکه cacheline هاشون همزمان `items` رو ادیت کنه و باعث اختلال در عمکرد همدیگه بشن. (بدون mutex در مثال بالا، ممکنه ۲ thread همزمان `items` رو ادیت کنن بدون اینکه تغییرات همدیگه رو ببینن و در نهایت `items` داخل DRAM خراب بشه.)

<div dir="ltr">

```diff
unsigned int takeItem(player& player, item& item)
{
-   static unsigned int totalItemsAdded = 0;
-   static unsigned int totalISpecialtemsAdded = 0;
+   static std::atomic_uint totalItemsAdded{0};
+   static std::atomic_uint totalISpecialtemsAdded{0};
    std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
-       totalISpecialtemsAdded++;
+       totalISpecialtemsAdded.fetch_add(1);
-   return totalItemsAdded++;
+   return totalItemsAdded.fetch_add(1);
}
```
</div>

در این مثال، `totalItemsAdded` به صورت atomic تعریف شده. این یعنی وقتی چند thread همزمان از این تابع استفاده میکنن، مقدار `totalItemsAdded` به درستی بین همه thread ها sync میشه و هیچ thread ای مقدار اشتباهی نمیبینه.

طرز عملکرد atomic ها اینطوریه که بجای اینکه مزاحم DRAM یشن، طبق استانداردهای تعریف شده و با تعامل با CPU های همسایه، cacheline خودشون رو از همدیگه کپی میکنن. نهایتا اگه شکست خورد مزاحم DRAM میشه و دیتا رو sync میکنه. استاندارد معروفی که برای این کار وجود داره، [MESI](https://en.wikipedia.org/wiki/MESI_protocol) هستش که با tag گذاری در cacheline ها باعث تعامل راحت و سریع بین CPU ها میشه.

## استفاده بهتر از cacheline
مموری ای که برای atomic ها استفاده میشه، فرقی با مموری های دیگه نداره. این فقط ++C هستش که برای راحت تر کردن کار، متغییر رو طوری نشون میده که انگار خود متغییر خاص هستش. درواقع صرفا از دستورات مربوط به atomicity در assembly استفاده میشه برای استفاده از اون قسمت مموری. 
این به این معنیه که مثل بقیه مموری ها در ++C، ممکنه بیشتر از یک متغییر در یک cacheline قرار بگیره. در مثال بالا، ممکنه `totalItemsAdded` و `totalSpecialItemsAdded` در یک cacheline قرار بگیرن و تغییر `totalItemsAdded` هر بار باعث sync شدن `totalSpecialItemsAdded` هم بشه. ممکنه نصف `totalItemsAdded` در یک cacheline باشه و نصف دیگش در یک cachline دیگه. برای جلوگیری از این مشکل، بهتره از `alignas()` استفااده کنیم تا مطمئن بشیم هر کدوم از متغییر های atomic فقط یک cacheline منحصر بفرد داشته باشن.

<div dir="ltr">

```diff
unsigned int takeItem(player& player, item& item)
{
-   std::static std::atomic_uint totalItemsAdded{0};
-   std::static std::atomic_uint totalISpecialtemsAdded{0};
+   using cachelineOffset = alignas(hardware_destructive_interference_size);
+   cachelineOffset std::static std::atomic_uint totalItemsAdded{0};
+   cachelineOffset std::static std::atomic_uint totalISpecialtemsAdded{0};
    std::lock_guard<std::mutex> lock(player.itemsMutex);
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
    if (item.isSpecial)
        totalISpecialtemsAdded.fetch_add(1);
    return totalItemsAdded.fetch_add(1);
}
```
</div>

</div>

