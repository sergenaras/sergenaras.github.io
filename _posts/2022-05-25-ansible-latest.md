---
title: Ansible Güncel Versiyon Kurulumu
date: 2022-05-25 12:00:00 -500
categories: [linux,automation]
tags: [linux,automation,ubuntu,debian]
---
Daha önce Ansible’ın ne olduğuna dair bir [makaleyi]() örnekleri ile birlikte yazmıştım. Orada bahsettiğim kurulum şekli Ubuntu veya CentOS üzerindeki son güncel hali olarak kurmaktadır. Ancak en güncel halini kurmak için bir kaç ekstra adım atmak gerekcek.

## Ubuntu / Debian

Öncelikle sistemimizdeki tüm paketlerin güncel olduğuna emin olmalıyız. Bunun için

```
$ sudo apt update
$ sudo apt upgrade -y
```

Tüm paketleri yükledikten sonra paket depoları hakkında özet bilgilerin bulunduğu software-properties-common paketini kuralım.

```
$ sudo apt install software-properties-common
Reading package lists... Done
Building dependency tree
Reading state information... Done
software-properties-common is already the newest version (0.99.9.8).
software-properties-common set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

Büyük olasılıkla bu paketin kurulu olduğunu göreceksiniz, şimdi sırada paket deposu eklemeye geldi.

```
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
Hit:1 http://archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://ppa.launchpad.net/ansible/ansible/ubuntu focal InRelease [18.0 kB]
Hit:3 http://archive.ubuntu.com/ubuntu focal-updates InRelease
Hit:4 http://archive.ubuntu.com/ubuntu focal-backports InRelease
Hit:5 http://archive.ubuntu.com/ubuntu focal-security InRelease
Get:6 http://ppa.launchpad.net/ansible/ansible/ubuntu focal/main amd64 Packages [1,132 B]
Get:7 http://ppa.launchpad.net/ansible/ansible/ubuntu focal/main Translation-en [756 B]
Fetched 19.9 kB in 1s (15.8 kB/s)
Reading package lists... Done
```

Komutun içerisinde update parametresini verdiğimiz için sonrasında apt update çalıştırmamıza gerek kalmadı. Şimdi Ansible indirebiliriz.

```
$ sudo apt install ansible
$ ansible --version
ansible [core 2.12.6]
  config file = /home/sergen/k8s-deploy/ansible.cfg
  configured module search path = ['/home/sergen/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/sergen/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Mar 15 2022, 12:22:08) [GCC 9.4.0]
  jinja version = 2.10.1
  libyaml = True
```

Böylece son versiyonu kurmuş olduk.

## RHEL / CentOS / Rocky Linux

Red Hat tabanlı işletim sistemlerinde öncelikle python paketlerinin yükseltilmesi gerekmektedir.

```
$ sudo dnf install python3 python3-pip -y
```

Sonrasında EPEL reposunu aktif hale getirmemiz gerekecektir.

```
$ sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

Sonrasında epel deposunu kullanarak ansible kurulumu yapalım.

```
$ sudo dnf install  --enablerepo epel  ansible
```

Böylece Red Hat tabanlı dağıtımlarda da son versiyona göre kurulum yapabildik.

```
$ ansible --version
ansible [core 2.12.2]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/sergen/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.8/site-packages/ansible
  ansible collection location = /home/sergen/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.12 (default, May 10 2022, 23:46:40) [GCC 8.5.0 20210514 (Red Hat 8.5.0-10)]
  jinja version = 2.10.3
  libyaml = True
```
