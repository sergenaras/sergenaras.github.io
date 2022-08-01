---
title: Wordpress
date: 2020-10-16 12:00:00 -500
categories: linux
tags: [linux, wordpress, centos 8, rhel 8]
---

WordPress dünya çapında en çok kullanılan web içerik yönetim panellerinden biridir. Üzerinde blog, kişisel web sitesi, e ticaret portalları oluşturulabilir. Bu yazıda daha önceki yazıda inşa ettiğimiz LEMP yapısı üzerine wordpress kurulumu yapacağız.

Bu sefer kurulumlara php katmanından başkayacağız. Önceki kuruluma ek olarak indirilmesi gerereken eklenti paketleri var.

```
# dnf install -y php-xmlreader php-curl php-exif php-ftp php-gd php-iconv  php-json php-dom php-simplexml php-xml php-mbstring php-posix php-sockets php-tokenizer
```

PHP ile olayımız yukarıdaki eklenti paketlerinin yüklenmesi ile bitiyor. Şimdi ngix tarafındaki konfigürasyon dosyamızı tazelememiz gerekecek.

```
# cat /etc/nginx/conf.d/sergenaras.blog.conf
server {
	listen 80; 
	server_name sergenaras.blog;

	root /var/web/sergenaras.blog/public_html/;

	index index.html index.php;

	access_log /var/web/sergenaras.blog/logs/access.log;
	error_log /var/web/sergenaras.blog/logs/error.log;

	# Don't allow pages to be rendered in an iframe on external domains.
	add_header X-Frame-Options "SAMEORIGIN";

	# MIME sniffing prevention
	add_header X-Content-Type-Options "nosniff";

	# Enable cross-site scripting filter in supported browsers.
	add_header X-Xss-Protection "1; mode=block";

	# Prevent access to hidden files
	location ~* /\.(?!well-known\/) {
		deny all;
	}

	# Prevent access to certain file extensions
	location ~\.(ini|log|conf)$ {
		deny all;
	}
        
        # Enable WordPress Permananent Links
	location / {
		try_files $uri $uri/ /index.php?$args;
	}

	location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	}
}
```

Konfigürasyon dosyasına uygun olarak ilgili dizinlerin oluşturulması gerekecek.

```
# mkdir -p /var/web/sergenaras.blog/public_html/
# mkdir -p /var/web/sergenaras.blog/logs/
```

Sonrasında bu konfigürasyonun doğruluğunu test etmek için nginx -t komutunu kullanabiliriz. Eğer çıktısı aşağıdaki gibiyse sorun yok demektir.

```
# nginx -t

nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Bu işlemleri tamamladıktan sonra servisleri yeniden başlatmamız gerekecek.

```
# systemctl restart nginx
# systemctl restart php-fpm
```

Bu aşamada Let’s Encrypt üzerinden siteye ssl alabilir ve bunu belirli aralıklarla yenilereyek sitemizin güvenli olduğuna dair kullanan kişiyi uyarabiliriz. Ancak bu aşama zorundu değil. Yani ssl almadan da devam edilebilir. Bundan ötürü bu aşamayı farklı bir kaynak olarak yazacağım.

Şimdi mariadb tarafında ilgili veritabanı ve kullanıcıları oluşturabiliriz.

```
# mysql -u root -p
> CREATE DATABASE wordpress;
> CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'wptest0123';
> GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
> exit
```

Açıklamak gerekirse, ilk satırada veritabanına yetkili kullanıcı ile giriş yaptık. Sonrasında “wordpress” adında bir veritabanı oluşturduk ve ardından wpuser adından bir kullanıcı oluşturup buna bir parola atadık. Dördüncü satırda ise bu oluşturduğumuz kullanıcıya “wordpress” veritabanı üzerindeki tüm yetkileri verdik ve çıktık.

Sırada /tmp dizinine gidip wordpress dosyalarını indirme ve sitenin asıl olarak çalışacağı dizin tarafındaki ayarları gerçekleştirilmesinde.

```
curl -O https://wordpress.org/latest.tar.gz
tar -zxvf latest.tar.gz
mv wordpress/* /var/web/sergenaras.blog/public_html/
```

Sırasıyla dosyayı indirdik, arşivden çıkarttık ve public_html içerisine taşıdık. Sırada wp-config.php dosyası içerisindeki yapılandırmalar var.

```
# cd /var/web/sergenaras.blog/public_html/
# cp wp-config-sample.php wp-config.php
# vi /var/web/sergenaras.blog/public_html/wp-config.php

/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wpuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'wptest0123' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

wp-config.php dosyası içerisinde yukarıdaki satırları veritabanı erişimi için gerektiği şekilde dolduruyoruz.

Tüm bu işlemlerden sonra dosyalardaki sahiplik yetkisini nginx üzerine atıyoruz.

```
chown -R nginx:nginx /var/web/
```

Artık web sitesine erişebiliriz. URL üzerinden ulaşabilirsiniz.

<img src="{{ 'assets/pic/2020-10-12-wordpress-01.png' | relative_url }}" />

Bu noktadan sonra yaptığımız ayarlar kullanıcı ile ilgili ayarlar olacak..

<img src="{{ 'assets/pic/2020-10-12-wordpress-02.png' | relative_url }}" />

<img src="{{ 'assets/pic/2020-10-12-wordpress-03.png' | relative_url }}" />


<img src="{{ 'assets/pic/2020-10-12-wordpress-04.png' | relative_url }}" />

İşlemleri tamamlayıp giriş yaptığımızda elimizde böyle bir panel olacak..

<img src="{{ 'assets/pic/2020-10-12-wordpress-05.png' | relative_url }}" />

URL çubuğundan sadece web sitesinin adresini girdiğimizde ise wordpress arayüzümüzü görebiliriz.

<img src="{{ 'assets/pic/2021-10-12-wordpress-06.png' | relative_url }}" />
































