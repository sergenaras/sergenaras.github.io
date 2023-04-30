# Linux Eat My Ram

Completed: January 21, 2023
Created by: Sergen
Created time: January 16, 2023 1:49 PM
Last edited by: Sergen
Last edited time: April 17, 2023 10:46 AM
Link: https://www.linuxatemyram.com
Status: Done
Tags: linux 102
Type: Article

**Neler oluyor?**

Linux kullanılmayan bellek alanından değerlendirmek için yakın zamanda kullanılan yazılımları bellek alanından tamamen silmez. Bundan ötürü `free` komutuyla bakıldığında bellek neredeyse tamamen kullanıyor gibi görünür ancak öyle değildir. Endişe edecek bir durum söz konusu değildir. 

**Neden böyle?**

Normalde bu işlem disk üzerinde gerçekleşir ancak bellek üzerinde gerçekleştiği durumda sistemin daha hızlı yanıt vermesini sağladığı için ve atıl durumdaki alanı kullanmak için bu şekilde kullanılmaktadır. Bu işe yeni başlayanların kafasını karıştırmak dışında bir dezavantajı bulunmamaktadır. 

Belleği herhangi bir şekilde aktif çalışan yazılımdan mahrum bırakmaz. 

**Çok fazla yazılım çalışıyorsa?**

Eğer daha fazla belleğe ihtiyaç varsa bu ön belleğe alma süreci önce azaltılarak, bellek üzerinde yer kalmadığı durumda da disk üzerine alınacak şekilde değişir. Disk üzerinde (swap alanı) ön belleğe alma zaten varsayılan olarak sistemlerde bulunan bir durumdur. 

**Daha fazla swap alanına ihtiyacım var mı?**

Hayır. Disk üzerinde ön belleğe alma işlemi sadece uygulama bellekte daha fazla alana ihtiyaç duymadığı durumda kullanılır. Swap alanını kullanmaz. Eğer yazılım daha fazla bellek alanı istiyorsa, ön belleğe alma işlemi, bellek üzerinden diske (swap alanına) kayar. 

**Linux’un bunu yapmasını nasıl durdurabilirim?**

Disk üzerinde ön belleğe alma işlemini durduramazsınız. İnsanlar bu işlemi kapatmak istemesinin en belirgin sebebinin bellek üzerindeki alanı işgal ettiğini düşünmeleri ancak bu düşünce doğru değil. Disk üzerinde ön belleğe alma sisteme yazılımların yüklenmesini ve daha hızlı çalışmalarını sağlar ama bellek üzerinden ayrılıp tamamen diskte çalışması söz konusu değildir. Bu yüzden kapatmanın bir anlamı yoktur.

Ancak, olası bir senaryoda bellekle alakalı şüphe ettiğiniz bir durum söz konusu olduğunda `echo 3 | sudo tee /proc/sys/vm/drop_caches` komutuyla buraya gidecek olan [paketlerin düşürülmesini](https://linux-mm.org/Drop_Caches) sağlayabilirsiniz. 

**Eğer kullanılmıyorsa; top ve free komutları neden tamamı kullanıldığını gösteriyor?**

Bu sadece terminoloji farkından kaynaklanıyor. Hem siz hem de Linux belleğin uygulamalar tarafından kullanıldığı (used) yahut kullanılmadığı durumda serbest (free) olduğu konuda hemfikirsiniz. 

Ama nasıl hem kullanılan hem de serbest alanın hesabı tutulabilir? Bunun için bir tablo var.

| Bellek | Siz | Linux |
| --- | --- | --- |
| yazılım tarafından kullanılan  | used | used |
| kullanılan ancak farklı kullanımlara açık | free (or available) | used (and available) |
| kullanılmayan | free | free |

Bazen bu alanlar “buffers” veya “cached” olarak da çağırılabiliyor. Bunları öğrendikten sonra hala bellekte alan olmadığında ısrarcı mısınız?

**Gerçekten ne kadar bellek alanı kullanıldığını nasıl görebilirim?**

Bunu anlamak için `free -m` komutunu çalıştırmalı ve `available` sütununa bakmalısınız. 

```bash
$ free -m
                total        used        free      shared  buff/cache   available
  Mem:           1504        1491          13           0         855      792
  Swap:          2047           6        2041
```

Görülebileceği üzere `used` yazan kısımda belleğin tamamı kullanılıyor gibi duruyor ancak `available` kısmına baktığınızda sistemin %47’sinin hala erişilebilir durumda olduğunu anlayabiliyoruz. 

Bu available kısmını teknik olarak daha detaylı incelemek isterseniz Linus Torvard’ın şu [commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34e431b0ae398fc54ea69ff85ec700722c9da773)’ine alalım.

**Ne zaman endişelenmeye başlamalıyım?**

Sağlıklı ve yeterli belleğe sahip bir linux sistemde beklenen ve zararsız durumlar:

- `free` sütununun 0’a yakın olması
- `used` sütunun `total` sütununa yakın olması
- `available` belleğin (veya `free + buffers/caches`) yeterli alana sahip olması (total’in %20’si denebilir.)
- `swap used` değerinin değişmemesi

Tehlike sinyali olabilecek ve düşük bellek durumunu anlayabileceğiniz durumlar:

- `available` belleğin (veya `free + buffers/caches`) 0’a yakın olması
- `swap used` sürekli azalıp artarak dalgalanması
- `dmesg -T | grep oom-killer` komutunu çalıştırdığımızda çıktı ile karşılaşmamız. Bellek yetersizliğinden yazılımların kapandığı anlamına gelir.

**Burada yazılanları nasıl doğrulayabilirim?**

Burada yazılanların ispatlarını görmek için [bu sayfaya](https://www.linuxatemyram.com/play.html) bakabilir ve disk üzerine ön belleğe alma işlemlerinin detaylarını uygulayarak görebilirsiniz.