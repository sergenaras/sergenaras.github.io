---
title: Firewalld
date: 2020-09-28 12:00:00 -500
categories: linux
tags: [linux,rhel,centos,rocky linux,networkmanager]
---
Firewalld pek çok linux dağıtımında kullanılan bir firewall yönetim çözümüdür. Firewalld üzerinden kural gruplarını yönetmek için bölge(zone) adında bir yapı kullanır. Bölgeler, basitçe bilgisayarın bağlı olduğu ağ ile olan güvenlik seviyesi ile kuralların nasıl trafik kuracağını belirlenmesini sağlar.

Gelen trafiği dağıtmak için firewalld iki yere bakar. Bunlardan ilki paketin geldiği **kaynak(source)** tanımlı mı? Tanımlı ise tanımlı olduğu yere göre ilgili kurala aktarılır. Diğeri ise paketin **ağ arayüzü(interface)** kısmıdır. Eğer bir ağ arayüzü tanımlanmış ise ilgili bölgenin kuralları uygulanır. Bu iki noktadan biri tanımlı değilse paket için **public** bölgesinin kuralları geçerli olacaktır.

Firewalld içerisinde varsayılan tanımlamaları ile birlikte tam 9 bölge bulunmaktadır. Bunlar:

**Public**, varsayılan bölgedir. Sisteme yeni eklenen her ağ arayüzü bu bölgeyi kullanır. SSH, NDNS ve Dhcpv6-client servislerine izin vermektedir.

**Home**, ev ağımız için kullanabileceğimiz bir bölgedir. Varsayılan izinleri public ile aynıdır.

**Work**, iş ağımız için kullanabileceğimiz bir bölgedir. Varsayılan izinleri public ile aynıdır.

**Dmz**, halka açık fakat ağdaki diğer kullanıcıların yetkilerinin sınırlı olduğu durumlar için tercih edilebilecek bir bölgedir. Varsayılan olarak sadece SSH servisine izin verir.

**Interal**, iç ağdaki servislerin birbiriyle iletişim kurmaları için tercih edilebilecek bir bölgedir.

**External**, dış ağdaki servislerle kendi servislerinizin iletişim kurmasını için tercih edilebilecek bölgedir.

**Trusted**, varsayılan olarak hiçbir servis, port ve kaynak tanımlı değildir. Tüm trafiğe izin verilen bir bölgedir.

**Block**, varsayılan olarak hiçbir servis, port ve kaynak tanımlı değildir. Tüm trafiğin engellendiği bir bölgedir. Engellediği pakete karşılık olarak IPv4 için “icmp-host-prohibited”, IPv6 için ise “icmp6-adm-prohibited” mesaj gönderir.

**Drop**, block ile aynı özelliklere sahiptir. Ekstra olarak block engellediği paketler için cevap dönerken drop karşı tarafa cevap göndermez.

|Komut              |	Açıklama                                                |
|-----------        |---------------------------------------------------------  |
|--list-all	        | Firewalld tarafında tanımlı her şeyin özetini sunar       |
|--list-all-zones    | Bölgeler ile ilgili bilgi verir.                          |
|--list-interfaces   | Ağ arayüzleri ile ilgili bilgi verir.                     |
|--list-port         | Portlar ile ilgili bilgi verir.                           |
|--list-services	    | Tanımlı servisleri ile ilgili bilgi verir.                |
|--get-services      | Sistemdeki tüm servisleri gösterir.                       |
|--get-zones         | Kullanılabilecek bütün bölgeleri gösterir.                |
|--get-default-zone  | Varsayılan bölgeyi gösterir.                              |
|--get-active-zones  | Aktif bölgeyi ve içerisidekileri gösterir.                |


Not: Firewalld içerisinde pek çok komut bulunmaktadır. Yukarıdakiler özet bir şekilde hızlıca bazı bilgilere ulaşmamızı sağlayacağı için listelenmiştir. Diğer komutlara ulaşmak için man dosyaları yahut –help komutundan faydalanılabilir.


**Masquerade**, bir ağ arayüzüne yönlendirilen paketlerin, o ağ arayüzünden çıkış yaptıktan sonra kaynak adresilerinin, çıkış yaptığı ağ arayüzü olarak gösterilmesine denir.

**Permanent**, yazdığımız komuttaki kuralın anlık değil, kalıcı olmasını sağlar. Bu parametrenin kullanılmadığı durumda reload yahut restart işlemi sonrasında kural kaybolacaktır.

**Reload&Restart**, ayarların geçerli duruma gelmesi için bu işlemlerden birini yapmamız gerekebilir.

### Olası Senaryolar

- Varsayılan bölgeyi değiştirmek için --set-default-zone=[zone] komutu kullanılır. Bu komut için –permanent komutuna ihtiyaç duyulmaz. Ayrıca varsayılan bölge /etc/firewalld/firewalld.conf dosyasından da değiştirilebilir.
```
firewall-cmd --set-default-zone=home
```


- Evdeki ağımıza sadece wlan0 ağ arayüzü üzerinden bağlandığımız bir senaryo olduğunu düşünelim. Ev içerisindeki tüm servislerin bu arayüz üzerinden çıkış yapması için ilgili ağ arayüzünü bölgeye nasıl atarız?
```
firewall-cmd --permanent --zone=home --add-interface=wlan0
```

- Aynı ağ arayüzünü bu bölgeden kaldırmak istersek de –add-interface komutu yerine –remove-interface komutunu yazmamız yeterli olacaktır. Peki ya bir ağ arayüzünün hangi bölge ile ilişkilendirildiğini nasıl öğrenebiliriz?
```
firewall-cmd --get-zone-of-interface=wlan0
```

- Belirli bir kaynağa ilişkin bölgeyi öğrenmek istersek:
```
firewall-cmd --get-zone-of-source=10.0.0.0./24
```

- Public bölgesinden gelen SSH bağlantılarının kesilmesini istediğimiz bir senaryo düşünelim. (SSHPort 22)
```
firewall-cmd --zone=public --remove-port=22
firewall-cmd --zone=public --remove-service=ssh
```

- Peki kendi cihazımındaki SSH iletişim portunu değiştirdiğimizi ve bu sayede dışarıdan gelen isteklerin direk düştüğü ama bir bölgeden gelen SSH isteğini doğru porta yönlendirmemiz gereken bir senaryo düşünelim.(Bu senaryoda localdeki SSH portu 44322 olarak belirlenmiştir.)
```
firewall-cmd –zone=external --add-forward port=port=22:proto=tcp:toport=44322
```

- Belirli bir kaynaktan gelen ve belirli port ve servislere ulaşması gereken bir kural yazılması gerek
```
firewall-cmd --permanent --zone=public --add-rich-rule='rule family=”ipv4” source address=”10.95.0.17/24” service name=”grafana” accept'
```

Trafiğe direk izin vermek yerine yukarıdaki işlemde farklı işlemler de yapabiliriz.

**Accept**, trafiği kabul eder

**Reject**, trafiği kabul etmez ve kabul etmediğine dair bilgi gönderir.

**Drop**, trafiği kabul etmez ve bilgi göndermez.