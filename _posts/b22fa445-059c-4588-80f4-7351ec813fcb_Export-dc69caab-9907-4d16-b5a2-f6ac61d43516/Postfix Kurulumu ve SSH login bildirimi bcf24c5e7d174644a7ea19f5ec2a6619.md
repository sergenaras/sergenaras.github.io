# Postfix Kurulumu ve SSH login bildirimi

Created by: Sergen
Created time: February 3, 2023 9:10 AM
Last edited by: Sergen
Last edited time: April 17, 2023 10:45 AM
Link: https://askubuntu.com/questions/179889/how-do-i-set-up-an-email-alert-when-a-ssh-login-is-successful
Status: Done
Tags: bildirim
Type: Article

- Postfix Kururulumuı
    
    ```bash
    apt install postfix
    ```
    
    Kurulum yaparken gelen ekranda İnternet seçeneği ile devam edip ekranda makine adı yazdığına dikkat edimesi gerekiyor. Bunu kontrol etmek için;
    
    ```bash
    $ cat /etc/postfix/main.cf | grep ^myhostname
    myhostname = ans-mgmt
    ```
    
    Sonrasında Google üzerindeki mail adresinden App Password almak gerekiyor.
    
    ![Ekran Resmi 2023-02-19 16.33.56.png](Postfix%20Kurulumu%20ve%20SSH%20login%20bildirimi%20bcf24c5e7d174644a7ea19f5ec2a6619/Ekran_Resmi_2023-02-19_16.33.56.png)
    
    ![Ekran Resmi 2023-02-19 16.34.26.png](Postfix%20Kurulumu%20ve%20SSH%20login%20bildirimi%20bcf24c5e7d174644a7ea19f5ec2a6619/Ekran_Resmi_2023-02-19_16.34.26.png)
    
    Bu aldığımız şifreyi `/etc/postfix/sasl` altındaki dizini altındaki `sasl_passwd` dosyasına yazmamız gerekiyor.
    
    ```bash
    $ cat /etc/postfix/sasl/sasl_passwd
    [smtp.gmail.com]:587 antikstdio@gmail.com:vqliwxgzrvvlogjy
    ```
    
    Sonrasında postfix için haritalama işlemi yapalım.
    
    ```bash
    postmap /etc/postfix/sasl/sasl_passwd
    ```
    
    Bu komutu çalıştırdıktan sonra aynı dizin altında veritabanı dosyası oluşacak
    
    ```bash
    $ ls /etc/postfix/sasl/
    sasl_passwd  sasl_passwd.db
    ```
    
    Güvenlik amacıyla dosyaların sahiplik ve yetkilerini değiştirelim.
    
    ```bash
    chown root:root /etc/postfix/sasl/sasl_passwd.db /etc/postfix/sasl/sasl_passwd
    chmod 0600 /etc/postfix/sasl/sasl_passwd.db /etc/postfix/sasl/sasl_passwd
    $ ls -ltr /etc/postfix/sasl/
    total 12
    -rw------- 1 root root    59 Feb 19 14:20 sasl_passwd
    -rw------- 1 root root 12288 Feb 19 14:21 sasl_passwd.db
    ```
    
    Bu aşamdan sonra relay ayarlarını değiştirelim. Bunun için `main.cf` içerisinde işlem yapacağız.
    
    ```bash
    vi /etc/postfix/main.cf
    relayhost = [stmp.gmail.com]:587
    ```
    
    Sonrasındaki aşağıdaki satırları da dosyanın sonuna ekliyoruz.
    
    ```bash
    smtp_use_tls = yes
    smtp_sasl_auth_enable = yes
    smtp_sasl_security_options =
    smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
    smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt
    ```
    
    Ekledikten sonra servisi yeniden başlatabiliriz.
    
    ```bash
    $ sudo systemctl restart postfix
    ```
    
    Test için aşağıdaki komutu çalıştırabliriz.
    
    ```bash
    mail -s "Test subject" recipient@domain.com
    ```
    
- SSH Login Bildirimi
    
    Bunun için PAM modülünü ve bir script kullanacağız.
    
    ```bash
    cat login-notify.sh
    #!/bin/sh
    
    # Change these two lines:
    sender="antikstdio@gmail.com"
    recepient="srgnaras@gmail.com"
    
    if [ "$PAM_TYPE" != "close_session" ]; then
    	host="`hostname`"
    	subject="SSH Login: $PAM_USER from $PAM_RHOST on $host"
    	# Message to send, e.g. the current environment variables.
    	message="`env`"
    	echo "$message" | mailx -r "$sender" -s "$subject" "$recepient"
    fi
    ```
    
    Bu dosyanın sahiplik yetkileri olarak root:root ve çalıştırma yetkisi veriyoruz.
    
    ```bash
    chown root:root login-notify.sh
    chmod 750 login-notify.sh
    # ls -ltr login-notify.sh
    -rwxr-x--- 1 root root 364 Feb 19 14:31 login-notify.sh
    ```
    
    Son olarak ise /etc/pam.d/sshd dosyasına aşağıdaki satırı ekliyoruz.
    
    ```bash
    vi /etc/pam.d/sshd
    session optional pam_exec.so seteuid /home/sergen/login-notify.sh
    ```
    
    Bunu yaptığımızda her SSH bağlantısında mail düşecek
    
    ![Ekran Resmi 2023-02-19 17.37.04.png](Postfix%20Kurulumu%20ve%20SSH%20login%20bildirimi%20bcf24c5e7d174644a7ea19f5ec2a6619/Ekran_Resmi_2023-02-19_17.37.04.png)
    
    Ancak bu şekilde içerideki kullanıcı geçişleri görünmüyor.