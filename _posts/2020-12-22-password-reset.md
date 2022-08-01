---
title: Parola Sıfırlama
date: 2020-12-22 12:00:00 -500
categories: [linux]
tags: [linux,password reset,centos 8,rhel 8]
---

Linux sistemlerde parola sıfırlama işlemi eğer ki makineye fiziksel erişiminiz varsa bir kaç komut yazarak bunu hızlı ve kolay bir şekilde halledebilirsiniz. Bu yazı hazırlanırken kullanılan ekran görüntüleri RHEL’den alınma olmasına rağmen aynı parola sıfırlama adımlarını CentOS ve Fedora’da da uygulayabilirsiniz.

Bu işlem için öncelikle sistemi yeniden başlatmalı ve açılışta kernel seçim ekranında e tuşuna basıp kernel betiğine erişim sağlıyoruz. O betik içerisindeki **linux** veya **linux16** ile başlayan satırı bulup sonuna **rd.break** komutu eklendikten sonra **CRTL+X** tuşları ile sistemi direkt betik içerisinden başlatıyoruz.


Betik içerisinde başladığımızda sistem bizi acil durum ekranına atacak. Burada öncelikle kök sisteme bağlanmamız gerekiyor.

```
mount -o remount,rw /sysroot/
chroot /sysroot/
```

Kök sisteme bağlandıktan sonra parolayı sıfırlamamız ve .autorelabel komutu ile SELinux’u yeniden etiketlememiz gerekmektedir.

```
passwd root
touch .autorelabel
exit
```

Kök sistemden bağlantımızı kopardıktan sonra tekrardan sadece okunabilir olarak tekrar bağlıyoruz ve acil durum ekranından çıkış yapıyoruz.

```
mount -o remount,ro /sysroot/
exit
```

Sistem yeniden başladığında artık yeni atadığımız parola ile sisteme giriş yapabiliriz.