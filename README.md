

# ارتباط بین میکروسرویس‌ها با استفاده از Redis و WebSocket

این راهنما توضیحاتی دربارهٔ چگونگی برقراری ارتباط بین میکروسرویس‌ها به کمک Redis و WebSocket را ارائه می‌دهد. این روش ساده اما کارآمد است و برای ارسال و دریافت پیام بین اپلیکیشن‌ها بدون استفاده از HTTP مناسب می‌باشد.

## میکروسرویس چیست 
مفهوم میکروسرویس به این ایده اشاره دارد که یک سیستم بزرگ را از طریق تقسیم آن به بخش‌های کوچکتر و مستقل، معمولاً به نام میکروسرویس‌ها، مدیریت کنیم. این میکروسرویس‌ها به صورت جداگانه اجرا می‌شوند و خودشان دارای داده، لایه وابستگی، و منطق تجاری مستقل خود هستند. در این راستا، برخی روش‌های ارتباطی درون سیستمی میان این میکروسرویس‌ها بدون استفاده از HTTP عبارتند از:

    Message Queue (MQ): در این رویکرد، میکروسرویس‌ها از طریق یک صف پیام با یکدیگر ارتباط برقرار می‌کنند. هر میکروسرویس می‌تواند پیامی را در صف قرار دهد و میکروسرویس دیگر آن را بخواند و پردازش کند. این روش ارتباطی برای مدیریت اتصالات آسنکرون به‌خوبی مناسب است.

    Event-Driven Architecture (EDA): در این حالت، میکروسرویس‌ها از طریق ارسال و دریافت رویدادها (Events) با یکدیگر ارتباط دارند. هر میکروسرویس می‌تواند به وقوع یک رویداد گوش کند و به آن واکنش نشان دهد. این رویکرد معمولاً با استفاده از یک سیستم پیام‌رسان ایجاد می‌شود.

    RPC (Remote Procedure Call): در این حالت، میکروسرویس‌ها به عنوان متد‌های فاصله دور برای یکدیگر فراخوانی می‌شوند. این روش شامل استفاده از پروتکل‌هایی مانند Protocol Buffers یا Apache Thrift می‌شود.

    Database Replication: ممکن است برخی از میکروسرویس‌ها به صورت مستقیم به دیگر میکروسرویس‌ها دسترسی پیدا کنند و اطلاعات را از پایگاه داده مشترک استخراج کنند. این رویکرد باید با دقت مدیریت شود تا هماهنگی و تنظیمات امنیتی متناسب باشد.

هر یک از این روش‌ها دارای مزایا و معایب خاص خود هستند و انتخاب بهترین رویکرد بستگی به نیازها و مشخصات پروژه خاص شما دارد.





## نصب و راه‌اندازی

### پیش‌نیازها
- Node.js
- Laravel
- Redis

### گام 1: راه‌اندازی Redis
پیشنهاد می‌شود ابتدا Redis را راه‌اندازی کنید. می‌توانید از [سایت رسمی Redis](https://redis.io/download) نسخهٔ مورد نظر خود را دانلود کنید و نصب کنید.

### گام 2: راه‌اندازی LaravelService در Node.js
1. وارد پوشه پروژه Node.js خود شوید.
2. اطمینان حاصل کنید که Redis نصب شده است.
3. فایل `LaravelService.js` را به پروژه خود اضافه کنید.
4. از کلاس `LaravelService` برای برقراری ارتباط با Laravel استفاده کنید.

```javascript
const EventEmitter = require("events");
const Redis = require("ioredis");
const Sub = new Redis();
const Pub = new Redis();
const { promisify } = require("util");
const uuid = require("uuid");
class LaravelService extends EventEmitter {
    constructor(action, data) {
        super();
        console.log("new LaravelService ...");
        this.response = null;
        this.channelName =
            process.env.LARAVEL_REDIS_CHANNEL ?? "laravel_database_general";
        this.data = data;
        this.action = action ?? "subscribed";
        this.configs = {};
        // ایجاد اشتراک بر روی کانال Redis
        Sub.subscribe(this.channelName, (err) => {
            if (err) {
                console.error("Subscription error:", err);
                return;
            }

            //  console.log(`Subscribed to channel: ${this.channelName}`);
        });

        // رویداد دریافت داده‌ها از کانال Redis
        Sub.on("message", (channel, message) => {
            this.parseResponse(message);
        });
        this.call(this.action, this.data);
    }

    async call2(action, data) {
        await Pub.publish(
            this.channelName + "_sender",
            JSON.stringify({ action, data, source: "nodejs" })
        );
    }

    call(action, data) {
        return new Promise((resolve, reject) => {
            const requestId = uuid.v4(); // ایجاد کد یونیک
            const requestData = { action, data, requestId, source: "nodejs" };
            const requestChannel = this.channelName + "_sender";

            const responseCallback = (channel, message) => {
                //console.log("Response received:", message);

                const response = this.parseResponse(message);
                if (response.requestId === requestId) {
                    // بررسی تطابق کد یونیک
                    Sub.removeListener("message", responseCallback);
                    resolve({ response, channel });
                }
            };
            Sub.on("message", responseCallback);

            Pub.publish(requestChannel, JSON.stringify(requestData), (err) => {
                if (err) {
                    reject(err);
                }
            });

            // تنظیم timeout برای 5 ثانیه
            const timeoutDuration = 5000; // 5 ثانیه
            setTimeout(() => {
                Sub.removeListener("message", responseCallback);
                reject(new Error("Timeout exceeded")); // ارسال خطای timeout
            }, timeoutDuration);
        });
    }

    parseResponse(message) {
        const data = JSON.parse(message);
        if (data && data.source && data.source !== "nodejs") {
            this.response = data;
            if (data.action == "configs") this.configs = data;
            this.emit("message", data);
        }
        return data;
    }
}

module.exports = LaravelService;

```
گام 3: راه‌اندازی SubscribeToGeneralChannel در Laravel
    ایجاد یک دستور آرتیزان جدید:
    

```

php artisan make:command SubscribeToGeneralChannel
```
ویرایش فایل دستور ایجاد شده (SubscribeToGeneralChannel.php) و کد زیر را اضافه کنید:

```php

// ...
use Illuminate\Support\Facades\Redis;

class SubscribeToGeneralChannel extends Command
{
    // ...

    public function handle()
    {
        Redis::connection('pubsub')->psubscribe(['*_sender'], function ($message) {
            // پردازش پیام دریافتی از Node.js
            echo $message;

            $messageArray = json_decode($message, true);

            if ($messageArray && isset($messageArray["action"])) {
                // انجام اقدامات مرتبط با درخواست‌ها
                $this->performActions($messageArray);
            }
        });
    }

    private function performActions($messageArray)
    {
        $response = ["source" => "laravel", "action" => $messageArray["action"], "requestId" => $messageArray["requestId"] ?? null];

        switch ($messageArray["action"]) {
            case "configs":
                // انجام اقدامات مرتبط با درخواست "configs"
                $mainConfig = new MainConfigsController();
                $configs = $mainConfig->index(true);
                $response = [...$configs, ...$response];
                Redis::connection('default')->publish('general', json_encode($response));
                break;
            // اقدامات مرتبط با سایر درخواست‌ها
            // ...
        }
    }
}
```
اجرای دستور برای شروع گوش دادن به کانال:

```bash

    php artisan redis:subscribe-general
```
استفاده

اکنون با اجرای دستورها و اضافه کردن اقدامات مرتبط با درخواست‌ها، میکروسرویس‌های شما قادر به تبادل اطلاعات و دستورات بین هم خواهند بود.

برای مثال، ارسال یک دستور از Node.js به Laravel:

```javascript

const LaravelService = require("./path/to/LaravelService.js");
const LaravelServiceInstance = new LaravelService("commandName", { foo: "bar" });
```


توجه داشته باشید که شما باید مسیر فایل `LaravelService.js` و سایر موارد  را به محل مناسب واقعی پروژه خود تغییر دهید.


همچنین نحوه ارسال دستور از لاراول به nodejs 

```php
 $resp=["source"=>"laravel","action"=>"configs","requestId"=>null,...$configs];
 Redis::connection('default')->publish('general', json_encode($resp));
```


مشارکت

اگر توضیحات یا اسکریپت‌ها نیاز به بهبود دارند، خوشحال می‌شویم که به این پروژه کمک کنید. لطفاً گزارش مسائل یا درخواست‌های ادغام خود را ارسال کنید. اگر هم کارتون راه افتاد لطفا یه ستاره منو مهمون کنید .
مجوز

MIT License



