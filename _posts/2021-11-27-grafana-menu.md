---
title: Grafana Menü ve İşlevleri
date: 2021-11-27 12:00:00 -500
categories: [linux]
tags: [linux,grafana,monitoring,ubuntu,debian]
---
Grafana farklı kaynaklardan aldıkları verileri işleyebildiği gibi kendi bünyesinde farklı ekiplerin hatta farklı organizasyonların çalışmasına da müsade etmektedir. Bu makale standart olarak kurulmuş bir Grafana içerisindeki menülerin detaylarını ve kullanıcı yönetimini ele almaktadır.

Grafana kurulumu sonrasında web arayüzüne ulaştığınızda aşağıdaki ekran ile karşılaşacaksınız. Bu ekranda bulunan bölümleri incelememiz gerekise sol tarafta bulunan menüler ana menülerimizidir. Bunlar üzerinden gösterge panelleri oluşturabilir, kullanıcıları yönetebilir, kaynak ekleyebilir ve eklediğimiz kaynakların çıktıları üzerinde denemeler gerçekleştirebiliriz. Sol altta bulunan kısımda kendi kullanıcımız ile ilgili ayarları yapabilir veya varsa organizasyonlar arasında geçiş yapabiliriz. Sağ üst köşede bulunan ikonlar ise “Home” adıyla gelen varsayılan gösterge panelinin ikonlarıdır. Detaylı olarak daha sonra anlatılacaklar.

<img src="{{ 'assets/pic/2021-11-27-grafana-menu-01.png' | relative_url }}" />

Sol tarafta bulunan menülerden detaylı oalrak ilerleyecek olursak, ilk başta bulunan ikon arama işlemleri için kullanacağımız ikondur. Burada göster panellerini isim ve tag’lerine göre arayabiliriz.


Bir altında bulunan seçenek ise gösterge paneli ve dosya eklemek için kullanacağımız kısımdır. Alt sekmeler olan “Dashboard” ile yeni gösterge paneli oluşturabilir, “Folder” ile gösterge panellerini gruplamak için dosya oluşturabiliriz. “İmport” kısmıyla ile Grafana sitesinde bulunan veya daha önceden yedek alınan gösterge panellerini içeri aktarabiliriz.


Burası dışarıdan Grafana içerisine eklediğimiz gösterge panelleri, paneller, oynatma listeleri ve ekran görüntülerini yönettiğimiz kısımdır.

Manage, burada gösterge panelleri liste halinde bulunur. Klasör oluşturularak gösterge paneller arasında gruplama yapılabilir.

Playlists, gösterge panellerini gruplayıp belirli aralıklarla değişecek şekilde slayt gibi dönmesini sağlayan bir mekanizma sunar. Bu sayede farklı gösterge panellerini tek bir ekranda görebiliriz.

Snapshots, hassas bilgi içeren gösterge panellerinin sorgu ve ayar kısımlarına erişimi engelleyerek sadece metriklerin görünmesini sağlar. Bunu üretmek için gösterge paneline girdiğimizde, en tepede “share” seçeneği içerisinden snapshot kısmına gitmemiz gerekmektedir. Oluşturduğumuz snapshot görüntülerini bu menüden yönetiriz.

Library panels, hazır paneller burada bulunur. Eğer birden fazla gösterge panelinde kullanacağınızı düşündüğünüz bir panel varsa buraya kayıt ederek “Create” aşamasında kütüphaneden seçerek işlerinizi hızlandırabilirsiniz.

<img src="{{ 'assets/pic/2021-11-27-grafana-menu-02.png' | relative_url }}" />


Bu kısım Grafana üzerindeki metriklerin durumlarını önizlemek için kullandığımız bir kısımdır. Aşağıda bununla ilgili bir örnek bulunmaktadır. Bu örnekte Grafana içerisinde varsayılan olarak gelen veriler kullanıldığı için farklı görselleştirme araçlarının kullanımını gösteremiyorum ancak

<img src="{{ 'assets/pic/2021-11-27-grafana-menu-03.png' | relative_url }}" />

Alarmları bu başlık altında ayarlarız. Burada iki alt sekme bulunmaktadır. Bu sekmelerden ilki gösterge panelleri içerisinde tanımlanmış olan alarmların durumlarını izlememizi sağlar, diğeri ise alarmların çalışacağı kanalların tanımlanacağı kısımdır.

Alarm ekleme ve yönetme işlerini gösterge panelleri ile birlikte detaylı olarak anlatmayı planlıyorum.

<img src="{{ 'assets/pic/2021-11-27-grafana-menu-04.png' | relative_url }}" />

Grafana Organizasyon bazında yönetici olunduğu durumda bu sekmeleri görmenize izin vermektedir. Burada bağlı olunan organizasyon ile ilgili eklentiler ekleyebilir, veri kaynakları tanımlayıp dışarıdan kullanıcı davet etme ve var olan kullanıcıları bu organizasyona dahil etme gibi seçenekleri kullanabilirsiniz.

Data Sources, organizasyon için geçerli olan veri kaynaklarını eklediğimiz kısımdır. Varsayılan olarak veritabanlarından, üçüncül parti yazılımlara kadar pek çok veri kaynağı içermektedir.

Users, organizasyon içerisindeki kullanıcıların tanımlandığı kısımdır.

Teams, organizasyon içerisinde farklı takımların birbirlerinin gösterge panellerini görmemesi gibi bir durum mevcutsa takımlar oluşturularak kullanıcılar bu takımlara eklenir ve gösterge panelleri bu takımlar içinde dağıtılır.

Plugins, burada Grafana içerisinde kullanılan eklentiler, görselleştirme bileşenleri ve üçüncül parti veri kaynağı bulunur/eklenebilir.

Preferences, organizasyon ile ilgili tanımların/ayarların yapıldığı kısımdır.

API Keys, API key tanımları buradan yapılır, yine yönetim işlemleri de bu sekmeden gerçekleştirilir.

<img src="{{ 'assets/pic/2021-11-27-grafana-menu-05.png' | relative_url }}" />

Bu menü Grafana Admin olarak yetkilendirilmiş kullanıcılara açıktır. Burada organizasyonların içerisindeki kullanıcılar ve organizasyonlar ile ilgili tanımlara müdehale edilebilir, server ile ilgili tanımlar kontrol edilebilir ve Grafana Server ile ilgili kullanım oraları incelenebilir.


Sol en altta bulunan bu menüde giriş yaptığımız kullanıcı ile ileili ayarları yapacağımız kısım bulunur. Ayrıca birden fazla organizasyon bulunduğu durumlarda organizasyonlar arasında geçiş işleri buradan gerçekleştirilir.












