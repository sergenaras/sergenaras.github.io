# Safer Bash Script

Author: Vaneyckt
Completed: January 8, 2023
Created by: Sergen
Created time: January 8, 2023 5:07 PM
Last edited by: Sergen
Last edited time: April 17, 2023 3:02 AM
Link: https://vaneyckt.io/posts/safer_bash_scripts_with_set_euxo_pipefail/
Status: Done
Tags: bash
Type: Article

Çoğu zaman yazılımcılar bash script yazmak ile bir yazılım dilinde program yazmayı eş değer tutarlar. Ancak bu yanlış bir düşünce tarzıdır. Programlama dillerinde bulunan koruma katmanları script’lerde varsayılan olarak bulunmamaktadır. Bir örnek vermek gerekirseİ go dili varsayılan olarak atanan herhangi bir değişken kullanılmadığında derleme aşamasında hata verecektir ancak script’ler bununla ilgili bir hata üretmez ve çalışmaya devam ederler. 

Bunu tanımlamak için bash bazı [builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html) kalıpları kullanabilir. Bunların arasında hataları önlemek için `set -euxo pipefail` öne çıkmaktadır. 

## Set -e

`-e` parametresi acil bir durumda script’in durmasını ve çıkış(exit) yapılmasını sağlar. Bu komut koşullu ifadelerdeki “false” çıktısının bir sorun olmadığını anlayacak kadar gelişmiştir. Eğer ekstra bir şeyler eklemek istenirse `|| true` ile farklı bir komut tetiklenebilir. 

```bash
$ cat sete1.sh
#!/bin/bash

foo
echo "bar"

$ ./sete1.sh
./sete1.sh: line 3: foo: command not found
bar
```

```bash
$ cat sete1.sh
#!/bin/bash

set -e

foo
echo "bar"

$ ./sete2.sh
./sete2.sh: line 5: foo: command not found
```

`set -e` komutunu eklediğimizde hatayı aldığımızda program çalışmayı bıraktı ve echo komutu işlemedi. 

```bash
$ cat sete3.sh
#!/bin/bash

set -e

$(ls foo)
echo "bar"

$ ./sete3.sh
ls: foo: No such file or directory

```

`$(ls foo)` komutu bir çıktı vermediği için sistem acil duruma düşüyor sonrasında çıkış yapıyor.

```bash
$ cat sete4.sh
#!/bin/bash

set -e

foo || true
$(ls foo) || true
echo "bar"

$ ./sete4.sh
./sete4.sh: line 5: foo: command not found
ls: foo: No such file or directory
bar
```

`|| true` etkisi ile birlikte bir hata olsa bile geçmesini sağladık ve programdan çıkmadı. 

```bash

$ cat sete5.sh
#!/bin/bash

set -e

if ls foobar; then
  echo "foo"
else
  echo "bar"
fi

$ ./sete5.sh
ls: foobar: No such file or directory
bar
```

## Set -o pipefail

Script normalde bir satırdaki son komuta bakarak çıkış koduna karar verir. Ancak bu bazı durumlarda istenmeyen durumlara neden olabilir. Bir `pipe ||` işareti bulunan satırda kontrolleri doğru yapabilmek için `-o pipefail` kullanılması gerekmektedir. 

```bash
$ cat set01.sh
#!/bin/bash

set -e

foo | echo "a"
echo "bar"

$ ./set01.sh
a
./set01.sh: line 5: foo: command not found
bar
```

Burada `pipe` işaretinin solunda `foo` komutu bir çıktı vermeyecek haliyle programı bozacak ancak sağ taraftaki `echo “a”` çıktı verecek.. bundan ötürü komutların hepsi geçiyor. 

```bash
$ cat set02.sh
#!/bin/bash

set -eo pipefail

foo | echo "a"
echo "bar"

$ ./set02.sh
a
./set02.sh: line 5: foo: command not found
```

`-o pipefail` eklentisi ile sadece o satır çalışıyor ve sonrasında duruyor. 

## Set -u

Bir değişken oluşturup kullanmadığınız durumda hata üreterek çalışmasını engeller. 

```bash
$ cat setu1.sh
#!/bin/bash

set-eo pipefail

echo $a
echo "bar"

$ ./setu1.sh
./setu1.sh: line 3: set-eo: command not found

bar
```

Hata verdi ancak diğer yandan script’i sonlandırmadı. 

```bash
$ cat setu2.sh
#!/bin/bash

set -euo pipefail

echo $a
echo "bar"

$ ./setu2.sh
./setu2.sh: line 5: a: unbound variable
```

Burada değişkene tanım yapılmadığı ile ilgili hata alıyoruz.

Eğer `${a:-$b}` durumunu kullanarak bir tanım yapmak isterseniz bu durumda `-u` parametresi durumu algılayıp tanımsız değişken için hata vermeyecektir. 

```bash
$ cat setu3.sh
#!/bin/bash
set -euo pipefail

DEFAULT=5
RESULT=${VAR:-$DEFAULT}
echo "$RESULT"

$ ./setu3.sh
5
```

Koşullu soruların içerisindeki değişkenler eğer atanmadıysa onu algılayabilir.

```bash
$ cat setu4.sh
#!/bin/bash
set -euo pipefail

if [ -z "${MY_VAR:-}" ]; then
  echo "MY_VAR was not set"
fi

$ ./setu4.sh
MY_VAR was not set
```

## Set -x

Bash script içerisindeki satırların çalışma zamanında set parametrelerinden geçip geçmediğini kontrol etmemizi sağlar. Satırların uygunluğunu ve çıktıları ekrana yansıtır.

```bash
$ cat setx1.sh
#!/bin/bash
set -euxo pipefail

a=5
echo $a
echo "bar"

$ ./setx1.sh
+ a=5
+ echo 5
5
+ echo bar
bar
```

## Set -E

Script içerisinde trap’ler ile çalışıldığı durumda çıktıyı ona göre şekillendirmeyi sağlar.

```bash
$ cat sete01.sh
#!/bin/bash
set -euo pipefail

trap "echo ERR trap fired!" ERR

myfunc()
{
  # 'foo' is a non-existing command
  foo
}

myfunc
echo "bar"

$ ./sete01.sh
./sete01.sh: line 9: foo: command not found
```

```bash
$ cat sete02.sh
#!/bin/bash
set -Eeuo pipefail

trap "echo ERR trap fired!" ERR

myfunc()
{
  # 'foo' is a non-existing command
  foo
}

myfunc
echo "bar"

$ ./sete02.sh
./sete02.sh: line 9: foo: command not found
ERR trap fired!
```

`-E` ile birlikte hata durumunda kendi çıktımızın ekrana çıkmasını sağladık. 

## Çözüm

Bu blog yazısı `set -euxo pipefail` veya `set -Eeuxo pipefail` komutlarını neden kullandığımız hakkında iyi bir fikir vermiştir.