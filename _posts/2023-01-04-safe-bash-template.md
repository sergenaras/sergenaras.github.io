# Safe Bash Template

Author: Maciej Radzikowski
Completed: January 8, 2023
Created by: Sergen
Created time: January 4, 2023 11:57 PM
Last edited by: Sergen
Last edited time: April 17, 2023 11:37 AM
Link: https://betterdev.blog/minimal-safe-bash-script-template/
Status: Done
Tags: bash
Type: Article

```bash
#!/usr/bin/env bash

set -Eeuo pipefail
trap cleanup SIGINT SIGTERM ERR EXIT

script_dir=$(cd "$(dirname "${BASH_SOURCE[0]}")" &>/dev/null && pwd -P)

usage() {
  cat << EOF # remove the space between << and EOF, this is due to web plugin issue
Usage: $(basename "${BASH_SOURCE[0]}") [-h] [-v] [-f] -p param_value arg1 [arg2...]

Script description here.

Available options:

-h, --help      Print this help and exit
-v, --verbose   Print script debug info
-f, --flag      Some flag description
-p, --param     Some param description
EOF
  exit
}

cleanup() {
  trap - SIGINT SIGTERM ERR EXIT
  # script cleanup here
}

setup_colors() {
  if [[ -t 2 ]] && [[ -z "${NO_COLOR-}" ]] && [[ "${TERM-}" != "dumb" ]]; then
    NOFORMAT='\033[0m' RED='\033[0;31m' GREEN='\033[0;32m' ORANGE='\033[0;33m' BLUE='\033[0;34m' PURPLE='\033[0;35m' CYAN='\033[0;36m' YELLOW='\033[1;33m'
  else
    NOFORMAT='' RED='' GREEN='' ORANGE='' BLUE='' PURPLE='' CYAN='' YELLOW=''
  fi
}

msg() {
  echo >&2 -e "${1-}"
}

die() {
  local msg=$1
  local code=${2-1} # default exit status 1
  msg "$msg"
  exit "$code"
}

parse_params() {
  # default values of variables set from params
  flag=0
  param=''

  while :; do
    case "${1-}" in
    -h | --help) usage ;;
    -v | --verbose) set -x ;;
    --no-color) NO_COLOR=1 ;;
    -f | --flag) flag=1 ;; # example flag
    -p | --param) # example named parameter
      param="${2-}"
      shift
      ;;
    -?*) die "Unknown option: $1" ;;
    *) break ;;
    esac
    shift
  done

  args=("$@")

  # check required params and arguments
  [[ -z "${param-}" ]] && die "Missing required parameter: param"
  [[ ${#args[@]} -eq 0 ]] && die "Missing script arguments"

  return 0
}

parse_params "$@"
setup_colors

# script logic here

msg "${RED}Read parameters:${NOFORMAT}"
msg "- flag: ${flag}"
msg "- param: ${param}"
msg "- arguments: ${args[*]-}"
```

## Bash Tercihi

```bash
#!/usr/bin/env bash
```

Script bir standart olan `shebang (#!)` ile başlıyor. Burada yapılabilecek en iyi referans `/usr/bin/env` içerisinden olacağı için tercih bu yönte kullanılmıştır. `/usr/bin/bash` kullanımı tavsiye edilmemektedir. Bunun sebebi `bash` her zaman `/usr/bin/bash` lokasyonunda olmayabilir. Ancak `/usr/bin/env` kullandığımızda lokasyon bağımsız `bash` çalıştırabiliriz.

## Başarısız Durumlarda İlerlememe

```bash
set -Eeuo pipefail
```

Script’e çalışma zamanında nasıl davranacağı ile ilgili işlem atamak için `set` kullanılır. Yukarıdaki komut ise bir işlem başarısız olduğunda script’in ilerlemesini engelleyip sonlandırır. Bir örnek üzerinden anlatmak gerekirse; elinizde eski bir konfigürasyon dosyası var. Onu silip yerine yenisini koyacaksınız. Bunu script üzerinde yaptığınız.. ancak çalıştırdığınızda `cp` komutu çalışmadı, dosyalar kopyalanmadı ama ardından gelen `rm` sebebiyle dosya silindi. Bu tarz bir durumun önüne geçmek için yukarıdaki komut kullanılır. Aksi halde script önceki komutların doğru çalışıp çalışmadığını kontrol etmez. 

## Script Lokasyonu

```bash
script_dir=$(cd "$(dirname "${BASH_SOURCE[0]}")" &>/dev/null && pwd -P)
```

Script içerisinde, script’in bulunduğu lokasyondaki dosyaları çalıştırmayı veya o lokasyona yeni dosya/log oluşturma gibi durumlarda kullanmak için ihtiyaç duyacağımız komuttur. 

Bu komutu kullanmadığımız durumda dosya yolunu elle yazmamız ve dosya yolu her değiştiğinde değiştirmemiz gerekecektir. 

## Temizlik İmandan Gelir

```bash
trap cleanup SIGINT SIGTERM ERR EXIT

cleanup() {
  trap - SIGINT SIGTERM ERR EXIT
  # script cleanup here
}
```

Script içerisinde yapılan işlemlerden düzgün bir şekilde çıkmak için kullanılan sistem çağrıları ve traplerden oluşan fonksiyondur. 

## Açıklama ve Yardım

```bash
usage() {
  cat << EOF # remove the space between << and EOF, this is due to web plugin issue
Usage: $(basename "${BASH_SOURCE[0]}") [-h] [-v] [-f] -p param_value arg1 [arg2...]

Script description here.

...
EOF
  exit
}
```

Bu sayede script’in neler yapabileceği kullanıcılara gösterebilir ve mini bir dokümantasyon oluşturmuş oluruz.

## Okunaklı Mesajlar

```bash
setup_colors() {
  if [[ -t 2 ]] && [[ -z "${NO_COLOR-}" ]] && [[ "${TERM-}" != "dumb" ]]; then
    NOFORMAT='\033[0m' RED='\033[0;31m' GREEN='\033[0;32m' ORANGE='\033[0;33m' BLUE='\033[0;34m' PURPLE='\033[0;35m' CYAN='\033[0;36m' YELLOW='\033[1;33m'
  else
    NOFORMAT='' RED='' GREEN='' ORANGE='' BLUE='' PURPLE='' CYAN='' YELLOW=''
  fi
}

msg() {
  echo >&2 -e "${1-}"
}
```

Script içerisinde kritik durumları kırmızı veya ön gereksinimleri geçen durumları yeşil ile göstermek gibi durumlar gerekebilir. Bu tarz durumlarda yukarıdaki gibi bir renk düzeni kullanmak işleri kolaylaştıracaktır. Tabi bu bir tercih meselesi.. 

`msg` ise sadece stdout ile verilen çıktıları değil, olur da bir hata çıkarsa onları da ekrana verebilmesi sağlayacaktır. 

Şu şekilde kullanılabilir. 

```bash
msg "This is a ${RED}very important${NOFORMAT} message, but not a script output value!"
```

## Parametreleri Ayıklamak

```bash
parse_params() {
  # default values of variables set from params
  flag=0
  param=''

  while :; do
    case "${1-}" in
    -h | --help) usage ;;
    -v | --verbose) set -x ;;
    --no-color) NO_COLOR=1 ;;
    -f | --flag) flag=1 ;; # example flag
    -p | --param) # example named parameter
      param="${2-}"
      shift
      ;;
    -?*) die "Unknown option: $1" ;;
    *) break ;;
    esac
    shift
  done

  args=("$@")

  # check required params and arguments
  [[ -z "${param-}" ]] && die "Missing required parameter: param"
  [[ ${#args[@]} -eq 0 ]] && die "Missing script arguments"

  return 0
}
```

Script’i parametrelerle çalıştırmak, birden fazla çalışma durumu eklemek veya birden fazla durumu bir arada çalıştırmak için çok kullanışlı bir yoldur.