---
title: Log Yönetimi
date: 2022-12-26 12:00:00 -500
categories: [linux]
tags: [linux]
---
Linux üzerinde logları yönetmek için kullandığımız bazı araçlar vardır.

## Rsyslog Nedir?

Linux sunucularda log toplamak için kullanılan bir yazılımdır. Genellikle merkezi log sunucusu oluşturmak için tercih edilir. Sistemdeki her olay syslog protokolü ile bu rsyslog üzerinden kayıt altına alınır. Şuan bir sektör standardı haline gelmiş ve hemen her dağıtımda aktif olarak kullanılmaktadır. Bunun en büyük nedeni sadece linux sistemler değil aynı zamanda windows ve switch, router gibi ağ çevresel birimlerinin de loglarını saklayabildiği için daha geniş bir kitleye hitap edebilmesi sayesindedir.

Syslog konfigürasyonu /etc/rsyslog.conf altında saklanır. Bunun yanı sıra özelleştirilmiş konfigürasyon dosyası oluşturmak isterseni /etc/rsyslog.d dizini altına konumlandırabiliriz.

### Syslog Tanımları ve Seviyeler

Sistem tarafında tanımlı syslog facity kaynakları şunlardır.

| Kaynak | Tanım |
| --- | --- |
| auth | giriş işlemleri |
| cron | zamanlanmış görevler |
| daemon | arkaplan hizmetleri |
| kernel | çekirdek mesajları |
| mail | mail kayıtları |
| syslog | syslog’un kendisi ile ilgili kayıtlar |
| lpr | yazıcı kayıtları |
| ftp | dosya trasfer kayıtları |
| local0 – local7 | kullanıcı tanımlı mesajlar |

Öncelik yani Severity/Priority sıralamaları ise şu şekilde (rule tanımındaki noktadan sonraki kısım)

| Severity | Priority |
| --- | --- |
| Emerg | 0 |
| Alert | 1 |
| Crit | 2 |
| Err | 3 |
| Warn | 4 |
| Notice | 5 |
| Info | 6 |
| Debug | 7 |

## Logrotate Nedir?

Log kayıtlarının kendi başına bıraktığınızda sistemler çalıştığı sürece dosya boyutları büyüyecek ve diski dolduracaktır. Bunu önlemek için log rotasyonu tanımı yaparız. Bu tanıma göre belirli aralıklarla bu dosyalar yeniden yazılır, sayısı artırılır azaltılır ve sistemdeki log oranını belirli bir boyut/tarih içerisinde tutarız.

Logrotate dosyası için belirli bir oluşturma standardı yoktur. Burada log alınacak yazılım veya systemin önem derecesini göre özelleştirilmiş konfigürasyonlar oluşturulabilir. Konfigürasyon dosyasını yazarken bilinmesi gereken bazı argümanlar vardır.

| Argüman | Tanım |
| --- | --- |
| daily | Log rotasyonunun her gün yapılacağını tanımlar. |
| weekly | Log rotasyonunun haftalık yapılacağını tanımlar. |
| monthly | Log rotasyonunun her ayın ilk günü yapılacağını tanımlar. |
| yearly | Log rotasyonunun her yılın ilk günü yapılacağını tanımlar. |
| rotate [adet] | Log dosyalarının kaç tane saklanacağı bilgisini burada veririz. |
| create | Log dosyası rotasyona uğradıktan sonra aynı isimle yeni bir tane oluşturulmasını sağlar. |
| nocreate | Rotasyon sonrasında aynı isimle yeni bir dosya oluşturulmasını engeller. |
| dateext | Bu seçenek ile dosyalarımız YYYY/MM/DD formatına göre arşivlenir. |
| dateformat | Dateext’in sunduğu tarih formatını kullanmak istemeyenler kendi formatlarını bu argüman ile belirtebilir. |
| compress | Rotasyon sonrasında dosyanın gzip ile sıkıştırılmasını sağlar. |
| missingok | Konfigürasyon için eklediğimiz log dosyası yoksa hata vermeden bir sonrakine geçer. Eğer bu seçeneği yazmazsak konfigürasyon için yazdığımız dosyalardan bir tanesi bile olmazsa logrotate hata verip rotate işlemini gerçekleştirmez. |
| olddir | Rotasyona uğrayan dosyaları bu argüman ile farklı bir dizine gönderebiliriz. |
| prerotate/endscript | Log rotation yapmadan önce yazmış olduğunuz bir scripti çalıştırır. |
| postrotate/endscript | Log rotation yaptıktan sonra yazmış olduğunuz bir scripti çalıştırır. |
| mail | Rotasyon sonrasında mail atmak için kullanılabilir. |
| size | Dosyaları belirtilen büyüklüğe geldiğinde rotasyona alır. |

### Konfigürasyon Tanımı

Logrotate için tanımlarımızı yapabileceğimiz bir dosya ve dizin bulunuyor. Linux işletim sistemlerinde varsayılan log rotasyonunu /etc/logratate.conf dosyasından görebilirsiniz. Sistem üzerinde çalışan yazılımların varsayılan dışına çıkan özel durumları varsa bunlar /etc/logrotate.d/ dizini altında konumlandırılır.

Örnek olarak bir dizinin log rotasyonuna uğramasını ele alalım. Bu rotasyon haftalık olarak 4 defa gerçekleşecek ve rotasyon sonrasında o günün tarihi ile dosyalar sıkıştırılacak. Bu süreçte eğer dizin içerisinde istenmeyen bir dosya varsa bu görmezden gelinecek.

```bash
$ mkdir /var/falancayazilim/
$ touch /var/falancayazilim/test.txt
```

Bu falancayazılım’a bir log rotasyonu hazırlamak istersek şu şekilde bir konfügrasyon dosyası oluşturabiliriz.

```bash
$ vi /etc/logrotate.d/falancayazilim.conf
/var/falancayazilim/*.txt {
	rotate 4
	weekly
	create
	compress	
	missingok
	dateformat -%Y%m%d
}
```

Tanımı yaptıktan sonra sonuçları görmek için elbet bir hafta beklememize gerek yok. `-d` `--debug` parametresi ile konfigürasyon dosyasını kontrol edebiliriz.

```bash
$  logrotate -d /etc/logrotate.d/falancayazilim.conf
```

Rotasyon yapmak istiyorsak ise `-f` `--force` parametresini kullanabiliriz.

```bash
$ logrotate -fv /etc/logrotate.d/falancayazilim.conf
```

not: `-v` parametresi verbose yani çıktıları ekrana basması için eklenmiştir.

## Journalctl nedir?

Linux sistemlerde sistem ve arka planda çalışan her programın log kayıtlarına bu komut üzerinden ulaşabiliriz. Journalctl’in en büyük faydalarından biri log kayıtlarını istenilen saat dilimine göre gösterebilmesi, bunun için sistemden **timedatectl** komutu ile tarih/saat ayarını yapmış olmanız gerekiyor.

Temel olarak **journalctl** komutu sistemdeki tüm logları önümüze serer. Buradan aradığımız özel bir kısım var ise **grep** komutu ile birlikte çalıştırdığımızda direk onlara ulaşabilir yahut **head**, **less** gibi komutlarla sadece baş ve sondan kesme işlemi yapabiliriz.

Son boot işleminden sonra oluşan logları görmek için **journalctl** komutuna **–b** parametresini eklememiz gerekir. Daha önceki loglar için ise parametrenin yanına ilgili boot kaydının referans değerini girerek çağırabiliriz. Geçmiş boot loglarını görmek, açılışta karşılaşılan bir sorun olduğunda log kayıtlarını karşılaştırıp durumu hakkında fikir yürütmemizi sağlayabilir.

Geçmiş bootları görüntülemek için kullanacağımız komut journalctl --list-boots komutudur. Bu komuttan önce sistem tarafında /var/log içerisine **journal** dizini oluşturulmalı ve **journalctl**’in konfigürasyon dosyasında saklama ayarları yapılmalıdır.

```bash
$ mkdir –p /var/log/journal
$ vi /etc/systemd/journald.conf
[Journal] alanı altındaki Storage alanının yanına persistent yazılmalı
```

Belirli zaman aralığında logları görmemiz de mümkün. Bunun için **since** 
ve **until** komutlarını kullanıyoruz. Gireceğimiz zaman dilimi sadece tarih veya tarih/saat olabilir. Format olarak **YYYY-MM-DD HH:MM:SS** şekildedir.

```bash
$ journalctl –since “01.03.2020 12:45:30” --until “03.03.2020 24:00”
$ journalctl –since yesterday
$ journalctl –since 10:00 –until “1 hour ago”
```

Servis durumlarını kontrol etmek için o alanda da filtreleme yapmamız gerekebilir. Bunun için `-u` parametresini kullanacağız.

```bash
$ journalctl –u nginx.service
```

Aynı zamanda bu parametreleri beraber de kullanabiliriz.

```bash
$ journalctl –u nginx.service --since 06:00
```

Servisleri kontrol etmenin yanı sıra bu servislerin oluşturduğu child processleri de kontrol edebiliriz. Bunun için **`_PID`** parametresini kullanırız. İlgili process’e ait PID değeri üzerinden filtreleme yapılabiliyor.

```bash
$ journalctl _PID=12555
```

Ayrıca belirli kullanıcılar ve grublar bazında bakmak istenirse `_UID` ve `_GID` komutlarıyla bu loglara da ulaşılabilir.

```bash
$ journalctl _UID=1000
```

Journal tarafında hangi user ve grupların loglandığı görmek için `-F` parametresini kullanmamız gerekiyor. Bu konuda daha fazla bilgi almak için man systemd.journal-fields komutu yeterli olacaktır.

```bash
$ journalctl –F _UID --> journal’da depolanan kullanıcı id’lerini getirir.
```

Patika vererek filtreleme yapmamız mümkün.

```bash
$ journalctl /usr/bin/bash
```

Kernelden gelen mesajları görüntülemek için `-k` yahut bu mesajlar dmegs içerisinde toplandığı için -dmegs komutunu kullanabiliriz.

```bash
$ journalctl –k 
$ journalctl –k –b 5 --> 5 boot önceki kernel mesajlarını okumamızı sağlar
```

Logları tek tek okumak ve ayıklamak yerine öncelik verip sadece o öncelik üzerindeki kayıtları okumamız mümkün. Bunun için `-p` parametresini kullanıyoruz.

```bash
$ journalctl –p err –b --> boot sırasındaki error ve üstü etikete sahip kısımları okumamızı sağlar.
```

Öncelik sırası için buraya tıklayın.

Yaptığımız log aramalarını json formatında çıktı olarak alabiliriz. Bunun için `-o` komutu ile json yazmamız yeterli. Hatta daha güzel formatta bir çıktı almak istiyorsak `json-prety` kullanabiliriz.

```bash
$ journalctl -b -u nginx -o json
```

| cat | Mesaj alanında kendisini gösterir. |
| --- | --- |
| export | Taşıma yahut yedeklemede kullanılabilecek binary format |
| json | Tek satırsa standart JSON dosyası |
| json-prety | Daha okunabilir JSON formatı |
| json-sse | Server-Sent ile uyumlu JSON formatı |
| short | Default syslog çıktı stili |
| short-iso | ISO 8601 wallclock zaman damgalı çıktı formatı |
| short-monotonic | Tekdüze zaman damgaları ile birlikte standart format |
| short-precise | Mikro saniyekesinliği ile default format |
| verbose | Girdi için uygun tüm journal alanlarını gösteren format |

Loglar üzerinde yapılan son işlemleri izlemek için `-n` komutu kullanılır. Bu komut tail komutu ile benzerlik gösterir.

```bash
$ journalctl –n
$ journalctl –n 20 --> son 20 kayıtı gösterir.
```

Devam eden logları takip etmek için `-f` komutunu kullanmamız yeterli, bu komut sayesinde akışı (flow) yakalayabiliriz.

```bash
$ journalctl -f
```

Logların disk üzerinde ne kadar alan işgal ettiğinini `--disk-usage` komutu ile inceleyebiliriz.

```bash
$ journalctl –-disk-usage
```

Logların büyümesi ile ilgili seçeneklere `/etc/systemd/journald.conf` doyasında ayar yapamız yeterli olacaktır.

**SystemMaxUse=:** Logların kaplayacağı maksimum disk alanını belirler

**SystemKeepFree=:** Loglar kayıt edilirken sistemde bırakılması gereken boş alan miktarıdır

**SystemMaxFileSize=:** Dosyalar rotate olmadan önce ilgili dosyaların ne kadar büyüyebileceğini belirler.

**RuntimeMaxUse=:** Geçici hafıza /run üzerinde kullanılabilir maksimum disk kapasitesini belirler.

**RuntimeKeepFree=:** Diğer kullanıcılar için geçici hafıza üzerine veri yazılırken ayrı tutulacak alanı belirler.

**RuntimeMaxFileSize=:** Geçici hafıza /run üzerinde yer alan log dosyasının rotate edilmeden önce ulaşabileceği maksimum kapasiteyi belirler.