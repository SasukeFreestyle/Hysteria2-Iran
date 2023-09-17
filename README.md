# Hysteria2-Iran
### Hysteria2 running on sing-box for bypassing internet censorship in Iran with TLS encryption.

DISCLAIMER: This Guide is machine translated.

****

- هدف اصلی این راهنما افزایش آگاهی در مورد چگونگی ساختن یکی درست است.
- فایل پیکربندی config.json اصلی اینجاست که شامل یک بلوک CIDR-IP درست است که سرور ارتباطی به سمت ایران برقرار نمی‌کند.

- این یک راهنمای مناسب برای مبتدی‌ها است، اما اگر شما کاربر تجربه‌ای لینوکس هستید، باید یک کاربر جدید بدون دسترسی sudo بسازید تا بتوانید hysteria2 را اجرا کنید و مجوزهای صحیح را به فایل‌ها اختصاص دهید.
- من می‌خواستم برای هر کسی که فنی نیست، ساختن یک سرور را بدون تغییر/ایجاد کاربران یا ویرایش مجوزهای فایل‌ها آسان کنم.
- همچنین تدریس خواهم کرد که چگونه از IP ایرانی‌تان برای ارتباط مستقیم با وب‌سایت‌ها/سرویس‌های ایرانی استفاده کنید بدون قطع اتصال "VPN" با استفاده از قوانین مسیردهی.

****

این راهنما برای Ubuntu 22.04 LTS نوشته شده است، اما هر توزیع مبتنی بر Debian هم باید کار کند.

### چه چیزهایی قبل از شروع این راهنما نیاز دارید. پیش‌نیازها
- یک VPS یا هر دستگاه دیگر / ماشین مجازی که Ubuntu 22.04 LTS یا یک توزیع مبتنی بر Debian را اجرا می‌کند.
- دسترسی SSH یا ترمینال/کنسول به سرور شما.
- نیاز دارید که نام کاربری‌تان را بدانید (نام کاربری که وقتی وارد Ubuntu می‌شوید را).
- پورت 443 در مسیریاب یا/و دیواره آتشین شما باز باشد.


****
## ابتدا نیاز به تنظیمات هسته برای بهبود عملکرد و افزایش محدودیت‌های ulimit داریم.

```
sudo nano /etc/sysctl.conf
```
این را در انتهای فایل کپی کرده و آن را ذخیره و ببندید.
```console
net.ipv4.tcp_keepalive_time = 90
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fastopen = 3
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
fs.file-max = 65535000
```

سپس این دستور را اجرا کنید تا فایل limits.conf را ویرایش کنید.
```
sudo nano /etc/security/limits.conf
```

این را در انتهای فایل کپی کرده و آن را ذخیره و ببندید.
```console
* soft     nproc          655350
* hard     nproc          655350
* soft     nofile         655350
* hard     nofile         655350
root soft     nproc          655350
root hard     nproc          655350
root soft     nofile         655350
root hard     nofile         655350
```

این دستور را اجرا کنید تا تنظیمات اعمال شوند.
```
sudo sysctl -p
```
## نصب Sing-box

ما قصد داریم که نرم‌افزار Sing-box را در یک پوشه به نام hy2 در پوشه خانه‌ی کاربری شما نصب کنیم. شما می‌توانید هر کجا که دلتان بخواهد در سیستم خود انتخاب کنید.

برای دیدن نام کاربری خود، دستور whoami را اجرا کنید.

```
SasukeFreestyle@web:~$ whoami
SasukeFreestyle
```
نام کاربری شما SasukeFreestyle است. حالا می‌توانیم نرم‌افزار Sing-box را در پوشه hy2 در پوشه خانه‌ی کاربری SasukeFreestyle نصب کنیم.

اگر نمی‌دانید کجا هستید، دستور pwd را اجرا کنید تا مسیر کامل خود را ببینید.
```console
SasukeFreestyle@web:~$ pwd
/home/SasukeFreestyle
SasukeFreestyle@web:~$
```
دستور pwd نشان می‌دهد که در حال حاضر در پوشه‌ی /home/SasukeFreestyle هستید.

حالا می‌توانیم یک پوشه به نام hy2 در پوشه‌ی خانه‌ی شما ایجاد کنیم.
```
mkdir hy2
```
برای تغییر به پوشه‌ی جدیدی که به نام hy2 ایجاد کرده‌ایم، دستور cd hy2 را اجرا کنید.

```
cd hy2/
```

دانلود آخرین نسخه از Sing-box.
لینک به صفحه انتشار.

https://github.com/SagerNet/sing-box/tags

در زمان نوشتن این متن، نسخه‌ی 1.5.0-beta.11 موجود است. برای دانلود فایل tar.gz مناسب برای سیستم خود، معمولاً نسخه‌ی linux-amd64 مناسب است. برای دانلود آن، از دستور wget استفاده خواهیم کرد.

```
wget https://github.com/SagerNet/sing-box/releases/download/v1.5.0-beta.11/sing-box-1.5.0-beta.11-linux-amd64.tar.gz
```
از بایگانی tar.gz فایل Sing-box را استخراج کنید.
```
tar -xzf sing-box-1.5.0-beta.11-linux-amd64.tar.gz --strip-components=1 sing-box-1.5.0-beta.11-linux-amd64/sing-box
```
کلیدهای OpenSSL را برای فایل پیکربندی ایجاد کنید. این فایل‌ها را بعداً استفاده می‌کنیم.
```
openssl ecparam -genkey -name prime256v1 -out ca.key

openssl req -new -x509 -days 36500 -key ca.key -out ca.crt -subj "/CN=google-analytics.com"
```
## نصب سرویس Sing-box
فایل سرویس ایجاد کنید.
```
sudo nano /etc/systemd/system/hy2.service
```

```console
[Unit]
Description=sing-box service
Documentation=https://sing-box.sagernet.org
After=network.target nss-lookup.target

[Service]
# به نام کاربری خود تغییر دهید. <---
User=USERNAME
Group=USERNAME
# به نام کاربری خود تغییر دهید. <---
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_SYS_PTRACE CAP_DAC_READ_SEARCH

#                       --->   به نام کاربری خود تغییر دهید.  <---
ExecStart=/home/USERNAME/hy2/sing-box -D /home/USERNAME/hy2/ run -c /home/USERNAME/hy2/config.json
#                       --->  به نام کاربری خود تغییر دهید.  <---
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```
بخش‌هایی که باید ویرایش شوند عبارتند از:
```console
User=USERNAME
Group=USERNAME
ExecStart=/home/USERNAME/hy2/sing-box -D /home/USERNAME/hy2/ run -c /home/USERNAME/hy2/config.json
```

مثال
```console
User=SasukeFreestyle
Group=SasukeFreestyle
ExecStart=/home/SasukeFreestyle/hy2/sing-box -D /home/SasukeFreestyle/hy2/ run -c /home/SasukeFreestyle/hy2/config.json
```


سرویس‌ها را مجدداً بارگذاری کرده و راه‌اندازی خودکار را فعال کنید.
```
sudo systemctl daemon-reload && sudo systemctl enable hy2
```

## پیکربندی Hysteria2

محتوای فایل config.json را از این مخزن کپی کنید یا با استفاده از دستور wget دانلود کنید.

```
wget https://raw.githubusercontent.com/SasukeFreestyle/Hysteria2-Iran/main/config.json
```

فایل config.json را ویرایش کرده و رمزهای عبور قوی را انتخاب کنید. همچنین باید مسیر برای فایل‌های openssl ca.key و ca.crt تولید شده را مشخص کنید.
```
nano config.json
```
یا
```
nano /home/USERNAME/hy2/config.json
```


بخش‌هایی که باید ویرایش شوند عبارتند از
```json
   "inbounds":[
      {
         "type":"hysteria2",
         "tag":"hy2-in",
         "listen":"::",
         "listen_port":443,
         "domain_strategy":"prefer_ipv4",
         "up_mbps":0,
         "down_mbps":0,
         "obfs":{
            "type":"salamander",
            "password":"password-one" <--- یک رمز عبور قوی انتخاب کنید.
         },
         "users":[
            {
               "name":"user",
               "password":"password-two" <--- یک رمز عبور قوی انتخاب کنید.
            }
         ],
         "ignore_client_bandwidth":true,
         "tls":{
            "enabled":true,
            "certificate_path":"/home/USERNAME/hy2/ca.crt", <--- مسیر به فایل ca.crt
            "key_path":"/home/USERNAME/hy2/ca.key" <--- مسیر به فایل ca.key
         }
      }
   ],
```

اکنون Sing-box را راه‌اندازی کنید و بررسی کنید که آیا Sing-box در حال اجراست یا خیر. باید نوشته "Active: active (running)" را مشاهده کنید.

```  
sudo systemctl start hy2 && sudo systemctl status hy2
``` 
```console
● sing.service - sing-box service
     Loaded: loaded (/etc/systemd/system/sing.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2023-09-17 17:53:39 CEST; 4h 19min ago
       Docs: https://sing-box.sagernet.org
   Main PID: 36087 (sing-box)
      Tasks: 13 (limit: 9347)
     Memory: 109.7M
        CPU: 3min 40.033s
             └─36087 /home/SasukeFreestyle/hy2/sing-box -D /home/SasukeFreestyle/hy2/ run -c /home/SasukeFreestyle/hy2/config.json
``` 
انجام شد! اکنون Sing-box Hysteria2 را اجرا خواهد کرد و به صورت خودکار فایل‌های geo-asset مورد نیاز را برای مسدود کردن اتصالات به ایران از سرور دانلود خواهد کرد.


## Client/Apps (Settings)

### NekoBox for Android
[NekoBox for Android 1.2.4+](https://github.com/MatsuriDayo/NekoBoxForAndroid/releases)

در نرم‌افزار NekoBox برای اندروید، روی علامت + کلیک کنید، سپس "نوع دستی [Hysteria]" را انتخاب کنید.
****
یک نام برای سرور خود انتخاب کنید.
نسخه پروتکل 2 را انتخاب کنید.
آدرس IP سرور خود را وارد کنید.
پورت 443 را تعیین کنید.
رمز عبور تاریک‌کردن (Obfuscation Password) را وارد کنید (password-one).
رمز عبور احراز هویت (Authentication Password) را وارد کنید (password-two).
گزینه "Allow Insecure" را فعال کنید.

***
## Routing rules
همیشه خوب است تنظیمات مسیریابی مناسبی انجام دهید تا تمام ترافیک خود را به سرور Hysteria2 ارسال نکنید. این کار همچنین به شما امکان می‌دهد تا وب‌سایت‌های ایرانی را در حالت فعال بودن VPN دیدن نمایید.

برو به تنظیمات.
Route را انتخاب کنید.
پایین برو و هم "Domain rule for Iran" و هم "IP rule for Iran" را فعال کنید.
به سرور خود مجدداً وصل شوید و یک وب‌سایت ایرانی را باز کنید.

***

### Nekoray (Windows/Linux)
[Nekoray 3.21+](https://github.com/MatsuriDayo/nekoray/releases)

در بخش "تنظیمات اصلی" (Basic Settings)، مقدار core را به sing-box تغییر دهید.

برای افزودن سرور به برنامه، روی سرور کلیک کنید، سپس پروفایل جدید را انتخاب کنید.
نوع را به Hystria2 تغییر دهید.
آدرس IP سرور و پورت 443 را وارد کنید.
رمز عبور تاریک‌کردن (Obfuscation Password) را وارد کنید (password-one).
رمز عبور احراز هویت (Authentication Password) را وارد کنید (password-two).
گزینه "Allow Insecure" را فعال کنید.
## Routing rules

به تنظیمات بروید و سپس تنظیمات مسیریابی (Routing Settings) را انتخاب کنید.
سپس گزینه مسیر ساده (Simple Route) را انتخاب کنید.
در بخش "IP" زیر "Direct" متن زیر را اضافه کنید.
```
geoip:private
geoip:ir
```
در بخش "Domain" زیر "Direct" متن زیر را اضافه کنید.
```
geosite:private
geosite:category-ir
```


## Credits

[@iSegaro](https://github.com/iSegaro) For the inital idea


[@bootmortis](https://www.github.com/bootmortis) for Iranian domain list and routing rules.

And many others.
