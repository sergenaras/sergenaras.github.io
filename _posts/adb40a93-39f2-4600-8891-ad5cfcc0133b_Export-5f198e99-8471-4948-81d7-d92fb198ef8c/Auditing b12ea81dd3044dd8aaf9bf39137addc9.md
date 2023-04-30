# Auditing

Completed: February 5, 2023
Created by: Sergen
Created time: December 6, 2022 1:30 AM
Last edited by: Sergen
Last edited time: April 17, 2023 11:04 AM
Status: Done
Tags: linux 101
Type: Research

Sistem üzerinde denetimi yapılması gereken iki kısım bulunmaktadır; ilki kullanıcı alanındaki yazılımlar ve araçlar; diğeri ise kernel alanındaki sistem çağrıları.. Kernel bileşeni kullanıcı alanındaki uygulamaların çağrılarını alır ve bunu `user`, `task` veya `exit` parametrelerine göre filtreleyebilir.

![audit_architecture.png](Auditing%20b12ea81dd3044dd8aaf9bf39137addc9/audit_architecture.png)

Kullanıcı alanındaki audit süreçleri kernel üzerinden bilgileri toplar ve bunları kullanarak bir log dosyası oluşturur. Diğer audit süreçleri için audit süreçleri için aşağıdaki komutları kullanırız:

- **audisp -** Bu araç audit daemon ile iletişime geçer ve olayları daha fazla işlemek üzere farklı yazılımlara yönlendirmek için kullanılır. Bu aracın amacı gerçek zamanlı analitik programlarına veri sağlamaktır.
- **auditctl** - Audit süreçlerinin işlemesi ile ilgili çalıştırılacak parametreler ve bunların kontrolü bu araç üzerinden gerçekleştirilir.
- **aureport** - Audit süreçlerine eklenmiş uygulamalar, araçlar ve sistem çağrılarının raporlarını görmek için kullanılmaktadır.
- Konfigürasyon
    
    Audit servisi `/etc/audit/auditd.conf` dosyası üzerinden konfigüre edilebilir. 
    
    ```bash
    # cat /etc/audit/auditd.conf 
    #
    # This file controls the configuration of the audit daemon
    #
    
    local_events = yes
    write_logs = yes
    log_file = /var/log/audit/audit.log
    log_group = root
    log_format = ENRICHED
    flush = INCREMENTAL_ASYNC
    freq = 50
    max_log_file = 8
    num_logs = 5
    priority_boost = 4
    name_format = NONE
    ##name = mydomain
    max_log_file_action = ROTATE
    space_left = 75
    space_left_action = SYSLOG
    verify_email = yes
    action_mail_acct = root
    admin_space_left = 50
    admin_space_left_action = SUSPEND
    disk_full_action = SUSPEND
    disk_error_action = SUSPEND
    use_libwrap = yes
    ##tcp_listen_port = 60
    tcp_listen_queue = 5
    tcp_max_per_addr = 1
    ##tcp_client_ports = 1024-65535
    tcp_client_max_idle = 0
    transport = TCP
    krb5_principal = auditd
    ##krb5_key_file = /etc/audit/audit.key
    distribute_network = no
    q_depth = 1200
    overflow_action = SYSLOG
    max_restarts = 10
    plugin_dir = /etc/audit/plugins.d
    end_of_event_timeout = 2
    ```
    
    Yukarıda Red Hat Enterprise Linux 8 üzerindeki varsayılan bir auditd konfigürasyonunu inceleyebilirsiniz.
    
    Burada kaç adet, ne boyutta log dosyası oluşabileceği; hangi durumlarda audit işleminin durdurulacağını ve ne seviyede işlem yapılacağı tanımlanmaktadır. Detaylı bilgi için [auditd conf](https://linux.die.net/man/8/auditd.conf) belgesi incelenebilir. 
    
- Servis
    
    Auditd diğer bileşenlerden farklı olarak `systemd` tarafından yönetilebilen bir parça değildir. Durumu incelenebilir, servis başlatılabilir ancak restart edilemez veya durdurulamaz. Bunun için standart `service` komutunu kullanarak işlem yapmak gerekmektedir. 
    
    ```bash
    service auditd start
    ```
    
    Aynı şekilde boot sürecinde başlamasını sağlamak için `systemd` yerine `chkconfig` kullanmak gerekemektedir. 
    
    ```bash
    chkconfig auditd on
    ```
    
- Kurallar
    
    Auditd kayıt altına aldığı logları belirli kural setleri üzerinden gerçekleştirir. Üç tip kural bulunmaktadır:
    
    - Kontrol kuralları - Audit sisteminin belirli konfigürasyonlara erişmesini ve değiştirebilmesini sağlayabilen kurallar
    - Dosya sistemi kuralları - Dosya sistemi üzerindeki dizin ve dosyaları izleyip onlara yapılan erişim ve düzenlemelerin izlenmesini sağlayan kurallar
    - Sistem çağrıları kuralları - Belirlenmiş bir programdan yapılan sistem çağrıları
    
    Audit kuralları `auditctl` komutuyla tanımlanabilir (bu durumda kurallar sadece yeniden başlatılana kadar geçerli olacaktır) veya `/etc/audit/audit.rules` dosyasına yazarak belirleyebiliriz. 
    
    - Kontrol Kurallarını Tanımlamak
        
        Kontrol kuralları için aşağıdaki tanımlanmış parametreleri kullanırız.
        
        - **-b** - Kernel üzerindeki mevcut audit için ayırılan tampon alanı arttırır.
            
            ```bash
            auditctl -b 8192
            ```
            
        - **-f** - Kritik bir hata tespit ettiğinde sistemin aksiyon almasını sağlar.
            
            ```bash
            auditctl -f 2
            ```
            
            Yukarıdaki komut kritik bir durumda “kernel panic” modunu tetikleyecektir.
            
        - **-e** - Audit servisinin enable/disable olmasını tanımlarız.
            
            ```bash
            auditctl -e 2
            ```
            
            Yukarıdaki komut Audit konfigürasyonunu kitleyecektir.
            
        - **-r** - Saniyede ne oranla mesaj oluşturulacağını tanımlarız.
            
            ```bash
            auditctl -r 0
            ```
            
            Yukarıdaki komut oluşturulacak konfigürasyonlarda mesaj limiti olmamasını sağlar.
            
        - **-s** - Audit sisteminin durumunu görmemizi sağlar.
            
            ```bash
            auditctl -s
            AUDIT_STATUS: enabled=1 flag=2 pid=0 rate_limit=0 backlog_limit=8192 lost=259 backlog=0
            ```
            
        - **-l** - Mevcut audit kurallarını listelememizi sağlar.
            
            ```bash
            auditctl -l
            LIST_RULES: exit,always watch=/etc/localtime perm=wa key=time-change
            LIST_RULES: exit,always watch=/etc/group perm=wa key=identity
            LIST_RULES: exit,always watch=/etc/passwd perm=wa key=identity
            LIST_RULES: exit,always watch=/etc/gshadow perm=wa key=identity
            ```
            
        - **-D** - Mevcut tüm audit kurallarını silmemizi salar.
            
            ```bash
            auditctl -D
            No rules
            ```
            
    - Dosya Sistemi Kurallarını Tanımlamak
        
        Dosya sistemi için kural tanımı yaparken aşağıdaki formatı kullanırız.
        
        ```bash
        auditctl -w path_to_file -p permissions -k key_name
        ```
        
        Buradaki;
        
        - **path_to_file** - audit edilmesi istenen dizin veya dosya
        - **permissions** - loglanacak izinlerin listesi
            - **r** - dosya veya dizinde okuma yetkilerini loglar
            - **w** - dosya veya dizinde yazma yetkilerini loglar
            - **x** - dosya veya dizinde çalıştırma yetkilerini loglar
            - **a** - dosya veya dizindeki değişiklikleri loglar
        - **key_name** - opsiyonel bir alandır, loglanacak olan dosyanın hangi isimle loglanacağını belirtir.
        
        Örnek;
        
        ```bash
        auditctl -w /etc/passwd -p wa -k passwd_changes
        ```
        
        Yukarıdaki komut ile tanım yapıldığında `/etc/passwd` dosyasındaki yapılan değişiklikleri kayıt altına alır.
        
        ```bash
        auditctl -w /sbin/insmod -p x -k module_insertion
        ```
        
        Yukarıdaki komut ile tanım yapıldığında ise sisteme bir modül eklendiğinde bunun audit loglaması yapılır. 
        
    - Sistem Çağrıları Kurallarını Tanımlamak
        
        Sistem çağrıları için kural tanımı yaparken aşağıdaki formatı kullanırız.
        
        ```bash
        auditctl -a action,filter -S system_call -F field=value -k key_name
        ```
        
        Burada;
        
        - **action,filter** - Burada virgül ile ayrılan ve iki değer tanımı yaparız. Bu tanımlar kuralın hangi durumlarda çalışacağını tanımlar. Bunlardan birisi `action`; `always` veya `never` olarak tanımlanır. Diğeri ise `filter` kısmındır ve burada `task`, `exit`, `user` ve `exclude` tanımlamaları yapılabilir.
        - **system_call** - belirlenen aksiyon ve filtrenin hangi sistem çağrılarına uygulanacağını buradan seçeriz.
        - **field=value** - ekstra olarak bir seçenek ekleyebilmemizi sağlar. Bu şekilde kuralı daha da kısıtlayabiliriz. Bu bir sistem mimarisi, grup ID, süreç ID veya benzer bir şey olabilir.
        - **key_name** - Opsiyonel bir alandır, kuralın hangi isimle loglanacağını belirlememizi sağlar.
        
        Örnek;
        
        ```bash
        auditctl -a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time_change
        ```
        
        Yukarıdaki komut adjtimex veya settimeofday sistem çağrılarından birisi çalıştığında 64x mimarisine sahip bir sistemse eğer çalışacaktır. 
        
         
        
        ```bash
        auditctl -a always,exit -F path=/etc/shadow -F perm=wa
        ```
        
        Sistem çağrılarını kullanarak dosya sistemi kuralı tanımlamak da mümkündür. Yukarıdaki örnekte `/etc/shadow` dosyasında `write,attiribute` çağrıları yapıldığında çalışacaktır. 
        
- Kaynaklar
    
    [7.5. Defining Audit Rules Red Hat Enterprise Linux 6 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/sec-defining_audit_rules_and_controls)
    
    [7.8. Creating Audit Reports Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-creating_audit_reports)
    
- Sorunlar
    
    Bu kısım altında sistemlerde karşılaşılan Audit sorunlarını ele alacağım.
    
    - “audit: backlog limit exceeded”
        
        Bu sorun ile arka plan loglarının mevcut sistemdeki kernel tampon limitinin üzerine çıktığı durumlarda karşılaşılır.
        
        Varsayılan `backlog_limit` değeri `320` olarak atanmıştır. 
        
        ```bash
        # auditctl -s
        enabled 1
        failure 1
        pid 991
        rate_limit 0
        backlog_limit 8192
        lost 0
        backlog 0
        backlog_wait_time 60000
        backlog_wait_time_actual 209
        loginuid_immutable 0 unlocked
        ```
        
        Bunu değiştirmek için `/etc/audit/rules.d/audit.rules` dosyasında bulunan `-b` parametresinin değeri değiştirilerek servis yeniden başlatıldığında sorun çözülecektir.
        
        Deneme amaçlı olarak sistemin yeniden başlatılması tavsiye edilmektedir. 
        
        Bu sorun Azure, AWS, Google Cloud tarafındaki makinelerde sıklıkla rastlanabilir. Makine kurulduğunda değeri arttırmak veya audit kullanılmıyorsa kapatmak en mantıklı çözüm çözüm olacaktır.
        
        [https://aws.amazon.com/premiumsupport/knowledge-center/troubleshoot-audit-backlog-errors-ec2/](https://aws.amazon.com/premiumsupport/knowledge-center/troubleshoot-audit-backlog-errors-ec2/)
        

- Kaynaklar
    
    [Linux Auditd for Threat Detection](https://izyknows.medium.com/linux-auditd-for-threat-detection-d06c8b941505)