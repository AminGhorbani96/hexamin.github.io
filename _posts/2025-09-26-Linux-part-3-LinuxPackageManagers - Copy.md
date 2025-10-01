---
layout: post
title: "قسمت سوم آموزش لینوکس - package managers"
date: 2025-09-26 14:00:00 +0330
categories: linux lpic threat-hunting
image: /assets/images/Deb_Sources.png
description: "در این قسمت میخواهیم درباره package manager ها صحبت کنیم"
tag: [packageManager, linux, yum, dnf, apt-get]
author: "amin Ghorbani"
---
## منابع نرم افزاری Repositories

در لینوکس شما زمانی که میخواهید یه برنامه ای را نصب کنید خیلی وقت ها نیاز نیست به سراغ فایل اجرایی اون برید و دانلود و مانند ویندوز با دکمه های NEXT اون رو نصب کنید. منابع نرم افزاری در اینترنت وجود دارند که سیستم عامل شما به آن ها متصل میشود و برنامه ای را که میخواهید نصب میکند، نکته باحالی که اینجا هست همونطوری که در قسمت قبل هم گفتیم برخی از این برنامه ها برای نصب نیاز به یکسری برنامه و کتابخانه دیگر دارند این Package Manager ها که در لینوکس وظیفه نصب برنامه ها از Repositoryها را برعهده دارند این نیازمندی ها را تشخیص میدهند و برای شما نصب میکنند.

هریک از توزیع های لینوکس منابع و Repository های خود را دارد و به نحوی آن را مدیریت میکند.
در توزیع های Debian Based ابزارهای apt-get, dpkg, apt برای مدیریت این منابع استفاده میشوند در این توزیع ها packageها پسوند deb.  دارند و از استاندارد نام گذاری زیر استفاده میکنند:

```bash
NameOfPackage-VersionOfPackage-Release-Arch.deb
```

اما در ماشین های debian base سیستم عامل کجا دنبال این منابع است ؟

```bash
cat /etc/apt/sources.list
```

اما در همین مسیر یک پوشه به اسم sources.list.d وجود دارد که شما میتوانید Repoهای خود را در آن قرار دهید. (به نظرتون مزیت این کار نسبت به اینکه در sources.list قرار بدیم مخازن رو چیست؟)



اما همه ما اگر یه بار خودمون یه لینوکس نصب کرده باشیم احتمالا دیدیم که درهمان مراحل اولیه به ما گفته اند دستور زیر را اجرا کنید:

```bash
sudo apt-get update
```

کار این دستور چیه؟ در حقیقیت این دستور به سراغ منابعی که ما به سیستم عامل به عنوان Repository معرفی کردیم میرود و لیست برنامه هایی که در این مخازن دارد را دریافت و Cache میکند ... تاکید میکنم تنها لیست برنامه ها !!! چرا اینکار اتفاق میافتد ؟ برای اینکه زمانی که ما با دستور زیر میخواهیم برنامه ای با نام firefox مثلا نصب شود ( اسم رو مثال زدم ممکنه اسم برنامه پسوند و پیشوند داشته باشه خیلی به این چیزاش گیر ندید :) ) به سراغ مخازنی که بهش معرفی کردیم میرود و از همان Cache متوجه میشود این برنامه در کدام مخزن در دسترس است تا بتواند آن را نصب کند

![تصویر دستور Deb_Sources](/assets/images/Deb_Sources.png)




```bash
sudo apt-get install firefox
```

زمانی که شما برنامه ای را شروع به نصب میکنید در ابتدا یه سری اطلاعات به شما میدهد که نیازمندی های این برنامه چیست؟ چقدر از Disk  شما قرار است اشغال شود، چه میزان دانلود باید انجام بدید و بعد از تایید شما یکی یکی نیازمندی ها دانلود و Unpack و نصب میشوند.
گاهی شما نمیخواهید برنامه را نصب کنید و تنها میخواهید آن را دانلود کنید در اینجا از دستور زیر استفاده میکنید

```bash
sudo apt-get download firefox
```

برای حذف برنامه هم میتونید حدس میزنید که دستور این است: 

```bash
sudo apt-get uninstall firefox
```
یادتون باشه با این دستور فقط برنامه حذف میشود اگر بخواهید تنظیمات هم حذف کنید باید از دستور purge استفاده کنید.

خب یه دستور دیگه هم زیاد دیدید:

```bash
sudo apt-get upgrade
```

این دستور به سراغ Cache ایجاد شده پس از apt-get update میرود، تمام برنامه های نصب شده شما را با Cache مقایسه میکند و اگر ورژن جدیدی از آن آمده باشد برنامه ها را بروزرسانی میکند.
دقت کنید این دستور به سراغ اینترنت نمیرود بلکه از Cache ایجاد شده استفاده میکند که ببنید نسخه جدیدی وچود دارد یا خیر.

خب یادتونه با دستور download گفتیم میتونیم deb. برنامه را دانلود کنیم؟ حالا همون فایل دانلود شده را با دستور dpkg میتوانیم نصب کنیم:

```bash
dpkg -i firefox.deb
```
اینجا اگر dependency لازم باشد دیگه اونو خودش نصب نمیکنه به شما خطا میدهد که بروید نصب کنید البته با دستور زیر میتونید dependency هایی که نیاز بودن و نصب نشدن رو نصب کنید

```bash
apt-get insall -f
```

حالا اگر لیست packageها نصب شده را بخواهیم چه کنیم؟

```bash
dpkg -l
```


اما خب حالا بریم سراغ توزیع های دیگه لینوکس که RedHat Baseها ازشون استفاده میکنن(RedHat- CentOs)

## دستور مدیریت YUM
 پکیج های ماشین های RedHat Base پسوند rpm. دارند، برای مدیریت Package ها در این نوع سیستم عامل ها از دستور yum استفاده میکنیم دستور dnf هم دستور جدید اینکار است.
اینجا هم اگر نیاز داریم Repoها را ببینیم باید به مسیر etc/yum.conf/ یا /etc/yum.repos.d/  سر بزنیم البته که در فدورا این مسیر وجود ندارد چرا که dnf دارد جایگزین yum میشود.

 ```bash
 cat /etc/yum.repo.d/fedora.repo
 ```
 
 ```result
 [fedora]
name=Fedora $releasever - $basearch
#baseurl=http://download.example/pub/fedora/linux/releases/$releasever/Everything/$basearch/os/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
enabled=1
countme=1
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
skip_if_unavailable=False

[fedora-debuginfo]
name=Fedora $releasever - $basearch - Debug
#baseurl=http://download.example/pub/fedora/linux/releases/$releasever/Everything/$basearch/debug/tree/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-debug-$releasever&arch=$basearch
enabled=0
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
skip_if_unavailable=False

[fedora-source]
name=Fedora $releasever - Source
#baseurl=http://download.example/pub/fedora/linux/releases/$releasever/Everything/source/tree/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-source-$releasever&arch=$basearch
enabled=0
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
skip_if_unavailable=False
 ```
 تو مسیر بالا میتونید Repo هایی که میخواهید را اضافه کنید اما همزمان این دستور معادل Update, upgrade در apt است.
 
```bash
yum update
```

یا مثلا زمانی که شما Install میزنید خودش میره اول update میکنه بعد نصب رو انجام میده

```bash
yum install firefox 
```
حالا اگر شما یه فایل rpm داشته باشید که بخواهید نصبش کنید کافیه از دستور زیر استفاده کنید

```bash
yum localinstall firefox.rpm
```

با دستور زیر هم میتوانید ببینید روی ماشین چه اتفاقاتی با yum رخ داده :

```bash
yum history
```



## دستور مدیریت rpm

برای اینکه فایل های rpm را مدیریت کنیم از دستور rpm استفاده میکنیم. برای نصب package از دستور زیر استفاده میکنیم

```bash
rpm -i firefox.rpm
```

دستور rpm سوییچ های مختلفی دارد :

```bash
-i install
-e erase : remove package
-U upgrade : if not installed install it if it installed upgrade it
-v verbose: write verbose log
-K check sign : check integrity and sign it
```

```bash
rpm -ev firefox
```

چون خود rpm این package رو نصب کرده پاکش میکنه :)

برای اینکه فایل های کانفیگ package نصب شده را پیدا کنید از دستور زیر استفاده میکنید:

```bash
rpm -qc firefox
```
 برای دیدن همه package های نصب شده میتوانیم از دستور زیر استفاده کنیم
```bash
rpm -qa
```

برای Extract کردن فایل های RPM میتوانیم از rpm2cpio استفاده کنیم:

```bash
rpm2cpio rpmname.rpm
cpio -idv < rpmname.cpio
```

اینجوری میتونیم محتوای داخلی rpm را ببینیم

