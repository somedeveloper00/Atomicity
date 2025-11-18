<div dir="rtl">

# کارکرد atomic در ++C در ۵ دقیقه

در این مقاله سعی میکنم در کمترین زمان ممکن، موضوع atomic یا atomicity در ++C رو از سخت‌افزار تا نرم‌افزار و روشهای استاندارد و پیشنهادی رو توضیح بدم.
## در سخت‌افزار چه اتفاقی میوفته؟

<div dir="ltr">

```cpp
void takeItem(player& player, item& item)
{
    player.items.push_back(item);
    std::invoke(onItemAdded, item);
}
```

</div>

<div dir="rtl">

پوینتر `player` و `items` کجان؟ پوینتر `this` یا `onItemadded` کجان؟ هر CPU طبیعتا دیتا رو از DRAM (همون رم عادی) میگیره و میذاره توی cache برای استفاده. توی cache، هر یک WORD میشه یک cacheline، که مثلا روی سیستمهای ۶۴ بیتی میشه ۶۴ بیت. وقتی یک دیتایی توی cache نیست و باید از DRAM گرفته بشه، بهش میگن cache-miss. هر cache-miss تقریبا ۱۰ برابر یک خوندن از cache طول میکشه. موقع cache-miss شدن، بحای اینکه همه ی cache با هم sync بشه، cacheline ها sync میشه طبق نیاز. یعنی در مثال بالا، وقتی `player`خونده میشه، اگر در cache وجود نداشته باشه، یک cacheline انتحاب میشه و پوینتر `player` از DRAM خونده میشه و در اون cacheline میشینه. این اتفاق برای `item` و همه دیتای موجود در هر قسمت از برنامه اتفاق میوفته. به همین دلیله که دیتا هایی که از هم فاصله کمتری دارن، cachemiss کمتری ایجاد میکنند و باعث افزایش سرعت برنامه میشن.


</div>

![image01.png](https://github.com/somedeveloper00/Atomicity/blob/main/res/image01.png?raw=true)    