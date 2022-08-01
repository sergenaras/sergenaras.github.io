---
title: Zabbix Web Arayüzüne SSL Eklemek
date: 2022-05-06 12:00:00 -500
categories: [linux,monitoring]
tags: [linux,monitoring,zabbix-server,zabbix web,ubuntu,debian]
---

SSL günümüzde bir ekstra güvenlik katmanından ziyade kesinlikle olması gereken, olmadığı durumda tarayıcılar tarafından güvensiz bulunup kullanıcıları uyaran bir hale geldi.

Tüm bu işlemlere başlamadan önce elimizde .crt ve .key uzantılı sertifikalarımızın olduğunu ve Zabbix’i Apache2 ile kurduğumuzu varsayıyorum.

Öncelikle elimizdeki sertifikaları Apache tarafından erişilebilir bir yere yerleştirmemiz gerekiyor. Bence en uygun yer

```
$ mkdir /etc/ssl/zabbix_cert/
```

Sonrasında elimizdeki sertifikaları buraya gönderelim.

```
$ cp certificate.key /etc/ssl/zabbix_cert/
$ cp certificate.crt /etc/ssl/zabbix_cert/
$ chmod -R 700 /etc/ssl/zabbix_cert
```

Bu aşamadan sonra makinemiz dışarıdan kendi IP’si ile çağırılacağı için hosts dosyasında ip ile domain tanımı olduğundan emin olalım.

```
$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 zabbix6

192.168.188.6  sergenaras.com

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Apache’nin varsayılan ssl konfigürasyon dosyası olan /etc/apache2/sites-available/default-ssl.conf dosyasında ServerName, ServerAlias, DocumentRoot ve SSL ayarlarını yapılandırmamız gerekmektedir. Bu ayarlar https ile çağırıldığında çalışacaktır.

```
$ cat /etc/apache2/sites-available/default-ssl.conf
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin administrator@sergenaras.com

                ServerName zabbix.sergenaras.com
                ServerAlias zabbix.sergenaras.com

                DocumentRoot /usr/share/zabbix

                # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
                # error, crit, alert, emerg.
                # It is also possible to configure the loglevel for particular
                # modules, e.g.
                #LogLevel info ssl:warn

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                # For most configuration files from conf-available/, which are
                # enabled or disabled at a global level, it is possible to
                # include a line for only one particular virtual host. For example the
                # following line enables the CGI configuration for this host only
                # after it has been globally disabled with "a2disconf".
                #Include conf-available/serve-cgi-bin.conf

                #   SSL Engine Switch:
                #   Enable/Disable SSL for this virtual host.
                SSLEngine on

                #   A self-signed (snakeoil) certificate can be created by installing
                #   the ssl-cert package. See
                #   /usr/share/doc/apache2/README.Debian.gz for more info.
                #   If both key and certificate are stored in the same file, only the
                #   SSLCertificateFile directive is needed.
                SSLCertificateFile      /etc/ssl/zabbix_cert/certificate.crt
                SSLCertificateKeyFile /etc/ssl/zabbix_cert/certificate.key

...
.
.
```

Sonrasında konfigürasyonun doğruluğunu kontrol edip Apache için varsayılan olarak atayabiliriz.

```
$ apache2ctl configtest
Syntax OK

$ a2ensite default-ssl
Enabling site default-ssl.
To activate the new configuration, you need to run:
  systemctl reload apache2

$ a2enmod ssl
Considering dependency setenvif for ssl:
Module setenvif already enabled
Considering dependency mime for ssl:
Module mime already enabled
Considering dependency socache_shmcb for ssl:
Enabling module socache_shmcb.
Enabling module ssl.
See /usr/share/doc/apache2/README.Debian.gz on how to configure SSL and create self-signed certificates.
To activate the new configuration, you need to run:
  systemctl restart apache2

$ systemctl restart apache2 
```

Şimdi varsayılan Apache sayfasında HTTP’den HTTPS’e yönlendirme için /etc/apache2/sites-available/000-default.conf dosyasına Rewrite komutları eklenmeli ve buranın asıl çağrıldığı nokta /var/www/html yolundan /usr/share/zabbix yoluna çevrilmelidir.

```
$ cat /etc/apache2/sites-available/000-default.conf
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin administrator@sergenaras.com
        DocumentRoot /usr/share/zabbix

        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>
```

Tüm bu işlemlerimiz bittiğinde a2enmod ile rewrite modunu aktif hale getiriyoruz.

```
$ a2enmod rewrite
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2
```

Tüm işlemleri tamamladığımıza göre artık servisi yeniden başlatıp tarayıcıdan kontrol edebiliriz.

```
$ systemctl restart apache2
```

<img src="{{ 'assets/pic/2022-05-06-zabbix-ssl-01.png' | relative_url }}" />
