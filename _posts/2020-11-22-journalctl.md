---
title: Jorunalctl
date: 2020-11-22 12:00:00 -500
categories: [linux, monitoring]
tags: [linux,monitoring,zabbix server,centos 8,rhel 8]
---

Red hat tabanlı sistemlerde sistem ve arka planda çalışan her programın log kayıtlarına bu komut üzerinden ulaşabiliriz. Journalctl’in en büyük faydalarından biri log kayıtlarını istenilen saat dilimine göre gösterebilmesi, bunun için sistemden **timedatectl** komutu ile tarih/saat ayarını yapmış olmanız gerekiyor.

Temel olarak **journalctl** komutu sistemdeki tüm logları önümüze serer. Buradan aradığımız özel bir kısım var ise **grep** komuyula birlikte çalıştırdığımızda direk onlara ulaşabilir yahut **head, less** gibi komutlarla sadece baş ve sondan kesme işlemi yapabiliriz.

Son boot işleminden sonra oluşan logları görmek için **journalctl** komutuna **-b** parametresini eklememiz gerekir. Daha önceki loglar için ise parametrenin yanına ilgili boot kaydının referans değerinigirerek çağırabiliriz. Geçmiş boot loglarını görmek, açılışta karılaşılan bir sorun olduğunda log kayıtlarını karşılaştırıp durumu hakkında fikir yürütmemizi sağlayabilir.

Geçmiş bootları görüntülemek için kullanacağımız komut **journalctl --list-boots** komutudur. Bu komuttan önce sistem tarafında **/var/log** içerisine **journal** dizini oluşturulmalı ve **journalctl**’in konfigürasyon dosyasında saklama ayarları yapılmalıdır.

```
mkdir -p /var/log/journal
vi /etc/systemd/journald.conf
```

**[Journal]** alanı altındaki **Storage** alanının yanına **persistent** yazılmalı

Belirli zaman aralığında logları görmemiz de mümkün. Bunun için **since** ve **until** komutlarını kullanıyoruz. Gireceğimiz zaman dilimi sadece tarih veya tarih/saat olabilir. Format olarak **YYYY-MM-DD HH:MM:SS** şekildedir.

```
journalctl --since yesterday
journalctl --since 10:00 --until "1 hour ago"
```

Servis durumlarını kontrol etmek için o alanda da filtreleme yapmamız gerekebilir. Bunun için **-u** parametresini kullanacağız.

```
journalctl -u nginx.service
```

Aynı zamanda bu parametreleri beraber de kullanabiliriz.

```
journalctl -u nginx.service --since 06:00
```

Servisleri kontrol etmenin yanı sıra bu servislerin oluşturduğu child processleri de kontrol edebiliriz. Bunun için **_PID** parametresini kullanıyoruz. İlgili process’e ait PID değeri üzerinden filtreleme yapılabiliyor.

```
journalctl _PID=12555
```

Ayrıca belirli kullanıcılar ve grublar bazında bakmak istenirse _UID ve _GID komutlarıyla bu loglara da ulaşılabilir.

```
journalctl _UID=1000
```

Journal tarafında hangi user ve grupların loglandığı görmek için -F parametresini kullanmamız gerekiyor. Bu konuda daha fazla bilgi almak için man systemd.journal-fields komutu yeterli olacaktır.

Journalctl –F _UID –> journal’da depolanan kullanıcı id’lerini getirir.

Patika vererek filtreleme yapmamız mümkün.

```
journalctl /usr/bin/bash
```

Kernelden gelen mesajları görüntülemek için -k yahut bu mesajlar dmegs içerisinde toplandığı için -dmegs komutunu kullanabiliriz.

```
journalctl -k
journalctl -k -b 5
```

5 boot önceki kernel mesajlarını okumamızı sağlar

Logları tek tek okumak ve ayıklamak yerine öncelik verip sadece o öncelik üzerindeki kayıtları okumamız mümkün. Bunun için -p parametresini kullanıyoruz.

```
journalctl –p err –b
```

boot sırasındaki error ve üstü etikete sahip kısımları okumamızı sağlar.
Öncelik sırası şu şekildedir.

|0|	Emerg   |
|1|	Alert   |
|2|	Crit    |
|3|	Err     |
|4|	Warning |
|5|	Notice  |
|6|	Info    |
|7|	Debug   |

Yaptığımız log aramalarını json formatında çıktı olarak alabiliriz. Bunun için -o komutu ile json yazmamız yeterli. Hatta daha güzel formatta bir çıktı almak istiyorsak json-prety kullanabiliriz.

```
journalctl -b -u firewalld -o json
```


|cat	            |Mesaj alanında kendisini gösterir.                     |
|export	            |Taşıma yahut yedeklemede kullanılabilecek binary format|
|json	            |Tek satırsa standart JSON dosyası                      |
|json-prety	        |Daha okunabilir JSON formatı                           |
|json-sse	        |Server-Sent ile uyumlu JSON formatı                    |
|short	            |Default syslog çıktı stili                             |
|short-iso	        |ISO 8601 wallclock zaman damgalı çıktı formatı         |
|short-monotonic	|Tekdüze zaman damgaları ile birlikte standart format   |
|short-precise	    |Mikro saniyekesinliği ile default format               |
|verbose	        |Girdi için uygun tüm journal alanlarını gösteren format|

Loglar üzerinde yapılan son işlemleri izlemek için -n komutu kullanılır. Bu komut tail komutu ile benzerlik gösterir.

```
journalctl -n
journalctl -n 20
```

son 20 kayıtı gösterir.
Devam eden logları takip etmek için -f komutunu kullanmamız yeterli, bu komut sayesinde akışı (flow) yakalayabiliriz.

```
journalctl -f
```

Logların disk üzerinde ne kadar alan işgal ettiğinini –disk-usage komutu ile inceleyebiliriz.

```
journalctl --disk-usage
```

Logların büyümesi ile ilgili seçeneklere /etc/systemd/journald.conf doyasında ayar yapamız yeterli olacaktır.

**SystemMaxUse=**: Logların kaplayacağı maksimum disk alanını belirler.

**SystemKeepFree=**: Loglar kayıt edilirken sistemde bırakılması gereken boş alan miktarıdır.

**SystemMaxFileSize=**: Dosyalar rotate olmadan önce ilgili dosyaların ne kadar büyüyebileceğini belirler.

**RuntimeMaxUse=**: Geçici hafıza /run üzerinde kullanılabilir maksimum disk kapasitesini belirler.

**RuntimeKeepFree=**: Diğer kullanıcılar için geçici hafıza üzerine veri yazılırken ayrı tutulacak alanı belirler.

**RuntimeMaxFileSize=**: Geçici hafıza /run üzerinde yer alan log dosyasının rotate edilmeden önce ulaşabileceği maksimum kapasiteyi belirler.