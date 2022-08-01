---
title: "Zabbix Medya Yönetimi: Telegram"
date: 2022-05-01 12:00:00 -500
categories: [linux,monitoring]
tags: [linux,monitoring,telegram,ubuntu,debian]
---
Zabbix üzerinde alarmları yönetmek için pek çok aracı medya yöntem kullanabiliriz. Bunların çok büyük bir kısmı Web Hook yapısını kullandığından ekleme yöntemleri birbirine benzer.. benim için alarmları yönetmek için en uygun ikili, mail ve telegram’dır.

Telegram tarafında bildirim işlemlerini otomatize hale getirmek için bot oluşturmamız gerekiyor. Bunun için Telegram içerisindeki BotFather’ı ve sonrasında IDBot’u kullanacağız.


<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-01.png' | relative_url }}" />

Öncelikle yeni bir bot oluşturmak için BotFather’da /newbot komutunu yütüretlim. Sonrasında bot’un ismini belirlememizi isteyecektir. Burada isim daha önce alınmışsa uyar verir ve benzersiz olana kadar devam eder.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-02.png' | relative_url }}" />

İsim kabul edildiğinde ise bot oluşmuş olur. Mesajda da dediği gibi bot için oluşturulan token size özeldir. Başkasının eline geçtiği durumda bot üzerinden yaptığınız her şey ele geçirenin de eline geçeceği için dikkat edilmelidir.

Bot’u oluşturduktan sonra yukarıdaki gibi yeni bir grup oluşturup IDBot’u ve oluşturduğumuz botu ekliyorum.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-03.png' | relative_url }}" />

Bu aşamada grubumuzda /getgroupid@myidbot komutu ile id bilgimizi alalım.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-04.png' | relative_url }}" />

Bu bilgiyi daha sonra Zabbix içerisinde kullanacağız. Ancak öncelikle Telegram medya ayarlarını yapmamız gerekiyor. Bunun için Administration > Media Types > Telegram olana gelip onu klonluyorum ve isminde değişiklik yapıp token kısmına botumuzu oluştururken aldığımız token bilgisini yazıyoruz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-05.png' | relative_url }}" />

Add dedikten sonra test edelim.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-06.png' | relative_url }}" />

Grub kısmındaki id bilgisini burada To kısmına ekliyoruz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-07.png' | relative_url }}" />

Test’in başarılı olduğuna göre bildirim ayarlarını yapabiliriz. Bunun için öncelikle Users > Admin > Media > Add üzerinden Admin kullanıcısına media ekleyelim.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-08.png' | relative_url }}" />

Bildirim ayarları için ise Configuration > Actions > Trigger Actions > Create Action üzerinden gidip hangi durumlarda çalışacağı ve ne işlem yapacağını belirtmemiz gerekiyor.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-09.png' | relative_url }}" />

Bu aksiyonun Host Grubu Linux Servers olanlarda tetikleneceğini belirliyoruz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-10.png' | relative_url }}" />

Tüm aşamalarda mesajları tüm aşamalarda ayarları yapılandırıyoruz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-11.png' | relative_url }}" />

Sonuç olarak aşağıdaki şekilde ayarları yapılandırmış oluyoruz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-12.png' | relative_url }}" />

Sonrasında gidip bir iki sunucumuzda agent servisimizi test için kapatıp açarsak eğer telegram tarafında aşağıdaki gibi bildirimleri görebiliriz.

<img src="{{ 'assets/pic/2022-05-01-zabbix-telegram-13.png' | relative_url }}" />











