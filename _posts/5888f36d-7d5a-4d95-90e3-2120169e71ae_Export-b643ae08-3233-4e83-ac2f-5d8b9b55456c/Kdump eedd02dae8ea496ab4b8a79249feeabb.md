# Kdump

Created by: Sergen
Created time: December 6, 2022 12:04 AM
Last edited by: Sergen
Last edited time: April 17, 2023 10:51 AM
Status: In progress
Tags: linux komutları

Kernel dump sunucu üzerinde kernel’in panic verdiği durumlarda yeniden başlaması ve yeniden başlatma işlemi öncesi bellek üzerindeki verileri bir vmcore dosyası olarak yazması durumu ve bu dosya üzerinden canlı sisteme bağlanır gibi sistemin incelenmesini sağlayan bir sistemdir.

## Red Hat

Bunu yapabilmek için Red Hat tarafında debug reposunun açık ve oradaki paketlerin erişilebilir olması gerekiyor.

Öncelikle kdump’ı indirelim.

```bash
$ yum install kexec-tools
```

Bunu indirdiğimizde sistemde `/etc/kdump.conf` adında bir dosya ve servis oluşacak. Bu dosya içerisinde konfigürasyon düzenlemesi gerçekleştirebiliriz. Dosya incelendiğinde varsayılan olarka gelen tüm ayarlar `#`(diyez) ile işaretlenmiş olarak görülebilir. Yeni bir tane eklenmesi durumunda eğer açıkta olan varsa kapatılması gerekecektir.

Burada ayarlanması gereken temel satırlar

```bash
$ vi /etc/kdump.conf
path /var/crash
core_collector makedumpfile -c
default reboot
```

Yapılabilecek diğer ayarlar; [buraya diğer ayarların anlamları eklenecek]

```bash
$ systemctl start kdump.service
$ systemctl enable kdump.service
```

Bu ayarı gerçekleştirdikten sonra grub üzerinde sistem yeniden başladığında bellekte kdump için yer ayrılmasını ayarlayalım. Bunun için /etc/default/grub üzerinde işlem yapacağız.

```bash
$ cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

Burada `GRUB_CMDLINE_LINUX` tanımında değişiklik yapacağız.

```bash
$ cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rd.lvm.lv=rhel/swap vconsole.font=latarcyrheb-sun16 rd.lvm.lv=rhel/root crashkernel=512M vconsole.keymap=us rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

Bu şekilde ilk açılışta sistemin 512 MB belleği ayarlamış oluyoruz ancak bunun kayıt olması için grub tarafına aktarılması ve sonrasında sistemin bir kez yeniden başlatılması gerekecektir.

```bash
$ grub2-mkconfig -o /boot/grub2/grub.cfg
$ grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
$ shutdown -r now
```

Aşağıdaki paketler debug işlemini yaparken kullanacağımız debug kernelleri indirmemizi sağlar.

### RHEL 7

```bash
$ subscription-manager repos --list | grep -i server-debug-rpms
$ subscription-manager repos --enable=rhel-7-server-debug-rpms
$ yum install kernel-debuginfo kernel-debuginfo-common
```

kernel ile uyumlu versiyonu indirmek için

```bash
$ yum install kernel-debuginfo-$(uname -r)
```

veya ihtiyaç olan versiyonu indirmek için

```bash
$ yum install kernel-debuginfo-3.10.0-327.36.2.el7
```

### RHEL 8

```bash
$ subscription-manager repos --enable=rhel-8-for-$(uname -m)-baseos-debug-rpms --enable=rhel-8-for-$(uname -m)-appstream-debug-rpms
$ yum install kernel-debuginfo-$(uname -r) kernel-debuginfo-common-$(uname -m)-$(uname -r)
```

Son olarak crash analizi yapabilmemiz için

```bash
$ yum install crash
```

Artık test işlemi için hazırız. Bunun için aşağıdaki komutu yürütüp bir kernel-panic oluşturacağız

```bash
$ **echo 1 > /proc/sys/kernel/sysrq ; echo c > /proc/sysrq-trigger**
```

Sunucu kapanacak ancak monitorda öncesinde şu ekran görülecek

![2022-12-05 22_06_43-Window.png](Kdump%20eedd02dae8ea496ab4b8a79249feeabb/2022-12-05_22_06_43-Window.png)

Burada kdump alma işlemi gerçekleştiriliyor. Sistem yeniden açıldığında dosyayı aşağıda görebiliriz.

```bash
$ ls -l /var/crash/
total 0
drwxr-xr-x. 2 root root 44 Jul  1 15:41 127.0.0.1-2022-07-01-15:37:57
```

Burada son olarak analiz aşamasına geçmek için crash ile işlem yapacağız

```bash
$ crash /var/crash/127.0.0.1-2022-07-01-15\:37\:57/vmcore /usr/lib/debug/lib/modules/3.10.0-1160.71.1.el7.x86_64/vmlinux

crash 7.2.3-11.el7_9.1
Copyright (C) 2002-2017  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.

GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

WARNING: kernel relocated [960MB]: patching 87430 gdb minimal_symbol values

crash: invalid kernel virtual address: 0  type: "possible"
WARNING: cannot read cpu_possible_map
crash: invalid kernel virtual address: 0  type: "present"
WARNING: cannot read cpu_present_map
crash: invalid kernel virtual address: 0  type: "online"
WARNING: cannot read cpu_online_map
crash: invalid kernel virtual address: 0  type: "active"
WARNING: cannot read cpu_active_map
WARNING: kernel version inconsistency between vmlinux and dumpfile

crash: invalid kernel virtual address: 0  type: "cpu_present_map"
crash: invalid kernel virtual address: 0  type: "cpu_present_map"
crash: cannot determine thread return address
WARNING: cannot determine pgdat list for this kernel/architecture

Segmentation fault
```

Burada neden segmentation fault alıyoruz. [araştırılacak]

```bash
$ crash --minimal /var/crash/127.0.0.1-2022-07-01-15\:37\:57/vmcore /usr/lib/debug/lib/modules/3.10.0-1160.71.1.el7.x86_64/vmlinux

crash 7.2.3-11.el7_9.1
Copyright (C) 2002-2017  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.

GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

WARNING: kernel relocated [960MB]: patching 87430 gdb minimal_symbol values

NOTE: minimal mode commands: log, dis, rd, sym, eval, set, extend and exit

crash>
```

Ancak bu durumda daha az komut kullanabiliyoruz. Kullanılabilen komutlar yukarıdakilerdir.

```bash
crash> help

*              extend         log            rd             task
alias          files          mach           repeat         timer
ascii          foreach        mod            runq           tree
bpf            fuser          mount          search         union
bt             gdb            net            set            vm
btop           help           p              sig            vtop
dev            ipcs           ps             struct         waitq
dis            irq            pte            swap           whatis
eval           kmem           ptob           sym            wr
exit           list           ptov           sys            q

crash version: 7.2.3-11.el7_9.1   gdb version: 7.6
For help on any command above, enter "help <command>".
For help on input options, enter "help input".
For help on output options, enter "help output".
```

Çalıştırılabilir tüm komutlar yukarıdakilerdir.

- Kaynaklar
    
    [https://access.redhat.com/solutions/9907](https://access.redhat.com/solutions/9907)
    
    [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/kernel_administration_guide/kernel_crash_dump_guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/kernel_administration_guide/kernel_crash_dump_guide)
    
    [https://www.linuxtechi.com/how-to-enable-kdump-on-rhel-7-and-centos-7/](https://www.linuxtechi.com/how-to-enable-kdump-on-rhel-7-and-centos-7/)
    
    [https://programming.vip/docs/example-using-crash-to-analyze-kdump-dump-kernel-crash-kernel.html](https://programming.vip/docs/example-using-crash-to-analyze-kdump-dump-kernel-crash-kernel.html)
    
    crash error
    
    [https://access.redhat.com/solutions/268653](https://access.redhat.com/solutions/268653)
    
    [https://access.redhat.com/solutions/719443](https://access.redhat.com/solutions/719443)
    
    [https://access.redhat.com/solutions/9907](https://access.redhat.com/solutions/9907)
    
    [https://access.redhat.com/solutions/210063](https://access.redhat.com/solutions/210063)
    
    [https://access.redhat.com/solutions/171713](https://access.redhat.com/solutions/171713)