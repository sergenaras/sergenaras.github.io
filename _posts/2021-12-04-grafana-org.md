---
title: Grafana Organizasyon Yönetimi
date: 2021-12-04 12:00:00 -500
categories: [linux,monitoring]
tags: [linux,grafana,monitoring,ubuntu,debian]
---
Grafana bize çok yönlü bir organizasyon yönetim paneli sunmaktadır. Burada bulunan yetkileri çok ince olarak denetleyemiyor olsak da kendi içerisindeki kalıplaşmış şema yeterli bir yönetim ortamı sağlamaktadır. Bu yazıda organizasyonları, takımları ve kullanıcıların nasıl yönetildiğine dair yapıda bahsedilecektir.

Grafana organizasyon yapısı ile kullanıcılara multi-tennant bir yapı sunmaktadır. Her bir organizasyon birbirinden farklı ve birbirinin bilgilerine erişemez durumdadır. O kadar ayrı bir yapı sunmaktadır ki server üzerine kurulan eklentileri bile ayrı ayrı aktif etmek gerekmektedir. Takım ve kullanıcı yapısını anlatabilmek için öncelikle bir organizasyonun nasıl açıldığını anlatmak istiyorum.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-01.png' | relative_url }}" />

Grafana panelinde “Server Admin” kısmında bulunan “Orgs” sekmesine tıkladığınızda yukarıdaki ekran gelecektir. Buradan genel olarak kaç tane organizasyona sahip olduğunuzu görebilir, organizasyonlardan herhangi birinin üzerine tıkladığımızda o organizasyona erişimi olan kullanıcıları görebilir ve üzerinde olmadığımız organizasyonları silebiliriz.

Bu aşamada organizasyon silerken dikkatli olmak gerekmektedir. İçerisindeki her şey ile birlikte siler.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-02.png' | relative_url }}" />

“New org” tuşuna tıkladığımızda yeni organizasyon oluşturmak için bir kısım gelir. Burada sadece isim vermemiz yeterlidir.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-03.png' | relative_url }}" />

Organizasyon oluştuktan sonra bizi o organizasyonun içerisindeki tercihler ekranına getirmektedir. Burada o organizasyon ile ilgili varsayılan ayarları değiştirebiliriz.

Organizasyonlarla ilgili yapacağımız şeyler bu kadar, eğer silmek istiyorsak ilk baştaki “Orgs” kısmından silinebilir. Ancak silindiği takdirde içerisindeki dashboard’lar ve paneller silinecektir.

Takımlar ise organizasyonların içerisindeki yetki ayrımlarını yapmak için kullanacağımız yapıdır. Bu sayede panellerin ve gösterge tablolarının yetkileri değiştirilebilir, alarmlar ve bildirimler kişiler yerine bu takımlara iletilerek kullanıcı karmaşası azaltılabilir.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-04.png' | relative_url }}" />

Takım oluşturmak için ilgili organizasyon içerisinde konfigürasyon kısmında takımlar sekmesine gelmemiz lazım. Bu kısımda aynı organizasyon sekmesinde olduğu gibi, aynı organizasyon içerisindeki takımları yönetebiliriz.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-05.png' | relative_url }}" />

“New team” kısmına geldiğimizde oluşturulacak takım için isim ve mail adresi tanımı yaptığımız ekran gelecektir.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-06.png' | relative_url }}" />

Oluşturduktan sonra kullanıcı ekleyebileceğimiz bir ekran gelecektir. Burada istediğimiz kullanıcıları ekleyebiliriz. Takımların yetkilerini kullandıkları yerlerde tanımladığımız için bu kısımda yapacağımız çok daha fazla bir şey bulunmamaktadır. 

<img src="{{ 'assets/pic/2021-12-04-grafana-org-07.png' | relative_url }}" />

Örnek olarak bir gösterge panelinde yetkiler kısmında yukarıdaki şekilde bir tanımlama yapılmıştır. Global olarak Editor ve View yetkisi olanlar görebilir, bunun yanı sıra Developer takımındakiler bu gösterge paneli üzerinde Admin yetkilerine sahiptir. System takımı ise sadece görebilir.

Global yetkileri devredışı bırakmak ve sadece gruplar üzerinden yürümek, birbiri ile iletişim kurmasını istemediğiniz ekipler olduğunda daha doğru bir yapı kurmanızı sağlayacaktır.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-08.png' | relative_url }}" />

Kullanıcıları anlatmamız gerekirse, en detaylı ayarların yapıldığı kısımdır. İki ayrı kısımdan erişilebilir, birisi Server Admin kısmındadır ve tüm organizasyonlardaki tüm kullanıcıları görebiliriz, diğeri ise Konfigürasyon sekmesindeki kısımdır ve orada sadece ilgili organizasyondaki kullanıcıları görebiliriz.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-09.png' | relative_url }}" />

Herhangi bir kullanıcının içerisine girdiğimizde yukarıdaki gibi bir ekran ile karşılacağız. Burada kullanıcının temel özellikleri, mail adresi gibi tanımlar yapılabilir.

“Permission” kısmında ise kullanıcının organizasyonlar üstü bir kullanıcı olup olmadığı tanımı yapılır. Örnek olarak gösterilen kullanıcımız Grafana’nın varsayılan admin kullanıcısı olduğu için Grafana Admin kısmı yes olarak bulunmaktadır.

“Organizations” kısmı ise organizasyonlarda nasıl yetkilerle erişeceği tanımını yaptığımız kısımdır.

<img src="{{ 'assets/pic/2021-12-04-grafana-org-10.png' | relative_url }}" />

Yeni bir organizasyon eklerken yukarıdaki ekran ile karşılaşacağız. Grafana üzerinde tüm yetkiler Admin, Editor ve Viewer’dan oluşmaktadır.

“Sessions” ise kullanıcını bağlantı geçmişini inceleyebileceğimiz bir alan sunmaktadır.

Grafana üzerindeki tüm yönetim aslında bu üç yapı parçasının çeşitli kombinasyonları üzerine kuruludur.


