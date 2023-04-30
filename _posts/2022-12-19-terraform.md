---
title: Terraform
date: 2022-12-19 12:00:00 -500
categories: [linux]
tags: [linux]
---

Günümüzde yapıların büyümesi, tekrarlanan iş yükünün artması ve insanlar tarafından yapılan işlerin dikkat eksikiği durumunda unutulan bir adım sebebiyle işlerin yürümemesi gibi durumları ortadan kaldırmak için işlerin otomatize hale gelmiş, yani sektördeki adıyla Infrastructure as a Code başlığı altında toplanmıştır. Bu başlık altında, yazılımların konteyner hale getirilmesinden(Docker); konteyner hale getirilen yazılımların organizasyonuna(Kubernetes); oradan konfigürasyon yönetimine(Ansible) ve nihayet Terraform’un ana konusu olan Infrastructure Providing’e kadar pek çok alanda bulunmaktadır.

Terraform, Hashicorp firmasının sunduğu ürünlerden birisidir. Temel amacı bulut servis sağlayıcıların hizmet sundukları yapıların elle değil de code ile oluşturulması ve yine yazılan kod temel alınarak silinebilmesidir. Ancak bulut servislerinin yanı sıra VMware gibi farklı tipteki kaynakların yönetimide sağlanabilmektedir.

- Kurulum
    
    Güncel bir kurulum için internetten araştırma yapılabilir. Ubuntu 20.04 için Terraform kurulumu aşağıdaki şekilde gerçekleştirilecektir.
    
    ```bash
    # curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add –
    # apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
    # apt install terraform
    $ terraform -version
    Terraform v1.1.9
    on linux_amd64
    ```
    
    Şuan terraform’un 1.1.9’un versiyonu kullanılmaktadır.
    
- Nasıl Çalışır
    
    Terraform’un çalışması için .tf uzantılı bir dosyaya [Hashicorp Configuration Language](https://www.terraform.io/language/syntax/) (HCL) ile yapılması istenilen işlemler yazılır ve bu dosya yürütülerek işlemleri yapması beklenir. Bu dosyaya genellikle `main.tf` adı verilir ve içerisinde ana olarak provider, resource, provisioner bileşenleri bulunmaktadır.
    
    ### Provider
    
    Terraform birden fazla altyapı kaynakları yaratabilir, güncelleyebilir ve silebilir. [Provider](https://www.terraform.io/language/providers) farklı tiplerdeki kaynakları yönetmek için gerekli API etkileşimini sağlamaktadır.
    
    Örnek üzerinden gitmek gerekirse, vmware’a bağlanması için bir provider tanımı yapalım.
    
    ```bash
    $ cat main.tf
    provider "vsphere" {
        user            = "sergen@vsphere.local"
        password        = "parolaparolaparola"
        vsphere_server  = "testvcenter.local"
    
        # If you have a self-signed cert
        allow_unverified_ssl = true
    }
    ...
    ```
    
    Provider kısmı bize terraform init komutunu çalıştırma hakkı kazandırır. Bu komut sayesinde yazdığımız kodun ihtiyaç duyacağı eklentiler main.tf dosyamızın bulunduğu yere indirilir.
    
    ```bash
    $ terraform init
    Initializing the backend...
    
    Initializing provider plugins...
    - Finding latest version of hashicorp/vsphere...
    - Installing hashicorp/vsphere v2.1.1...
    - Installed hashicorp/vsphere v2.1.1 (signed by HashiCorp)
    
    Terraform has created a lock file .terraform.lock.hcl to record the provider
    selections it made above. Include this file in your version control repository
    so that Terraform can guarantee to make the same selections by default when
    you run "terraform init" in the future.
    
    Terraform has been successfully initialized!
    
    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.
    
    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
    $ ls 
    main.tf  .terraform  .terraform.lock.hcl
    ```
    
    ### Resource
    
    Terraform ile ilgili en önemli tanımlar resource ile yapılır. Her resource bloğu bir veya birden fazla altyapı bileşen ve servisine ait tanımlamalar gerçekleştirir.
    
    Yine bir örnek üzerinden gitmek gerekirse Vmware üzerinde bir `folder` oluşturalım.
    
    ```bash
    $ cat main.tf
    ...
    data "vsphere_datacenter" "dc" {}
    
    resource "vsphere_folder" "parent" {
        path = "VMworld"
        type = "vm"
        datacenter_id = data.vsphere_datacenter.dc.id
    }
    ```
    
    Burada öncelikle `data` kısmından bahsetmek istiyorum. Resource gibi o da bir Terraform bileşenidir ve bilgi tutmak için kullanılır. Bu sayede resource kısmında `datacenter_id` argümanında olduğu gibi oradan gelen bilgileri işaret ederek her seferinde veri çekmesini önleyebiliriz.
    
    Resource kısmında bir folder oluşturmak istediğimizi belirtiyoruz. Burada bir `path` belirtmediğimiz için (yani direkt klasörün adını yazdığımız için) ve tip olarak `vm` seçili olduğu için `VM` and `Template` sekmesinde en dış katmana bir klasör oluşturacaktır.
    
    Bu aşamada kodumuzu çalıştırmanın iki kısmı bulunuyor. Birincisi `terraform-plan`.. bu sayede kodun neler yapacağını görebiliriz. Diğeri ise `terraform apply`, kodun gerçekten çalışmasını sağlar.
    
    ```bash
    $ terraform plan
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    
    Terraform will perform the following actions:
    
      # vsphere_folder.parent will be created
      + resource "vsphere_folder" "parent" {
          + datacenter_id = "datacenter-3"
          + id            = (known after apply)
          + path          = "terraform-test"
          + type          = "vm"
        }
    
    Plan: 1 to add, 0 to change, 0 to destroy.
    ─────────────────────────────────────────────────────────────────────────────────────
    Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
    ```
    
    Plan ile birlikte mevcut altyapı ile kodumuz çalıştığında altyapıda oluşacakları hesaplayıp bize gösteriyor.
    
    Bu aşamada bir yandan da dosyaları izlersek terraform-plan ile birlikte dosyalarımızın arasında bir de `.tfstate` uzantılı dosya katıldığını göreceğiz. Terraform ile işlem yaptıkça bu dosya güncellenecektir. Eğer yazılan kod ile ayağa kaldırılmış bir şeyler varsa elle temizlemek yerine `terraform-destroy` ile kaldırılması önerilir aksi halde bu dosyanın silinmesi gerekecektir. Diğer türlü koddaki durum ile gerçek durum eşleşmediği için hata alacağız.
    
    <aside>
    💡 Bu aşamadan önce `terraform-init` çalıştırılmazsa çıktı şöyle olacaktır.
    
    </aside>
    
    ```bash
    $ terraform plan
    Error: Inconsistent dependency lock file
    The following dependency selections recorded in the lock file are inconsistent with the current configuration:
    - provider registry.terraform.io/hashicorp/vsphere: required by this configuration but no version is selected
    To make the initial dependency selections that will initialize the dependency lock file, run: terraform init
    ```
    
    Oluşturduğumuzda ise benzer bir ekran göreceğiz.
    
    ```bash
    $ terraform apply
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    Terraform will perform the following actions:
      # vsphere_folder.parent will be created
      + resource "vsphere_folder" "parent" {
          + datacenter_id = "datacenter-3"
          + id            = (known after apply)
          + path          = "terraform-test"
          + type          = "vm"
        }
    Plan: 1 to add, 0 to change, 0 to destroy.
    Do you want to perform these actions?
      Terraform will perform the actions described above.
      Only 'yes' will be accepted to approve.
      Enter a value: yes
    
    vsphere_folder.parent: Creating...
    vsphere_folder.parent: Creation complete after 2s [id=group-v187]
    
    Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
    ```
    
    [https://lh4.googleusercontent.com/6YWl2KmOSYWNIa2kMJEHphPpx4137IwWGEi8Y7su1XFOiGKkOQwQfVzZjCgxA4-FGuESKXWlRSAAMZIThvHUyVzuIZofvuPgnsnTWQ5M8Kz3yrMbtLE9P1nB5Wnp6NOa1sltra4uBrdez575DGuugG4Qh6gcM7NoDDakV_XfvNvmSQe3x3teD6Wuisfpeq3S0SC5_yq5iQ](https://lh4.googleusercontent.com/6YWl2KmOSYWNIa2kMJEHphPpx4137IwWGEi8Y7su1XFOiGKkOQwQfVzZjCgxA4-FGuESKXWlRSAAMZIThvHUyVzuIZofvuPgnsnTWQ5M8Kz3yrMbtLE9P1nB5Wnp6NOa1sltra4uBrdez575DGuugG4Qh6gcM7NoDDakV_XfvNvmSQe3x3teD6Wuisfpeq3S0SC5_yq5iQ)
    
    Plan komutu ile hemen hemen aynı çıktı ancak burada ekstra olarak bir değişiklikleri kabul sorusu ve sonrasında oluşturma aşaması bulunuyor.
    
    ```bash
    $ terraform apply
    vsphere_folder.parent: Refreshing state... [id=group-v187]
    
    Note: Objects have changed outside of Terraform
    
    Terraform detected the following changes made outside of Terraform since the last "terraform apply":
    
      # vsphere_folder.parent has changed
      ~ resource "vsphere_folder" "parent" {
          + custom_attributes = {}
            id                = "group-v187"
          + tags              = []
            # (3 unchanged attributes hidden)
        }
    
    Unless you have made equivalent changes to your configuration, or ignored the relevant attributes using ignore_changes, the following plan may include actions to undo or respond to these changes.
    ─────────────────────────────────────────────────────────────────────────────────────
    No changes. Your infrastructure matches the configuration.
    Your configuration already matches the changes detected above. If you'd like to update the Terraform state to match, create and apply a refresh-only plan:
      terraform apply -refresh-only
    Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
    ```
    
    ### Provisioner
    
    Altyapı için gerekli kaynakları yarattıktan sonra bir script veya shell komutu çalıştırmak için provisioner kullanılır.
    
    ```bash
    $ cat main.tf
    ...
    provisioner "remote-exec" {
        
         connection {
            type     = "ssh" // for Linux its SSH 
            user     = "sergen"
            password = “parola”
            host     = "${vsphere_virtual_machine.cloned_virtual_machine.default_ip_address}"
          }
        
        inline = [
          "echo parola | sudo -S apt-get -y update",
          "sudo apt-get -y install nginx",
          "sudo service nginx start",
        ]
      }
    }
    ...
    ```
    
    Bu kısım genellikle kodun son kısımlarında bulunur ve oluşturma aşamasından sonra sunucuya yapılacak iç müdehalelelerin tanımlandığı kısımdır. Komut yürütme, dosya kopyalama gibi işlemler yapılabilir. Genellikle tanımlanan scriptler çalıştırılarak yapılmak istenilen işlemler gerçekleştirilir.
    
    [https://lh5.googleusercontent.com/fjFVzEzazbw1_TzCrU9m4tJ_ZqShph3jGnKLL8nzqdMh0pqjvgpvNZ5EsRCsUjYCTLiozQacStfLEyme1lZrGS8ZScTse6DE6qvtoTiMbuu8jM2cLb2O-d_X-YBVThEnPsQ3S7OBFlwWNBEOVsV9JBU6EIgWkDmbGZg62VUenP5j4DD7MBpEFixdj5yFo-imP5MEpLV-6g](https://lh5.googleusercontent.com/fjFVzEzazbw1_TzCrU9m4tJ_ZqShph3jGnKLL8nzqdMh0pqjvgpvNZ5EsRCsUjYCTLiozQacStfLEyme1lZrGS8ZScTse6DE6qvtoTiMbuu8jM2cLb2O-d_X-YBVThEnPsQ3S7OBFlwWNBEOVsV9JBU6EIgWkDmbGZg62VUenP5j4DD7MBpEFixdj5yFo-imP5MEpLV-6g)
    
    Burada dikkat edilmesi gereken bir husus olarak root olmayan bir kullanıcı ile işlem yapılacağı zaman sudo yetkisi olmasına ve yukarıdaki gibi komut satırı içerisine parolanın verildiğine emin olmak gerekecektir. Aksi halde yapılması istenilen işlem parola girilmediği için askıda kalacak ve tamamlanamayacaktır.
    
    [https://lh4.googleusercontent.com/UuBU2c3HKveL_42znHMRO-hDkK3qP1C1efMFnNmsj-tkM407_KYPM4SprhrEXg_9OQgkYGh6LgzzasRq8aoRkapePqQTO00OIIQnMBgn8hBtcZo7LMXP6rlyOxKXPEILGlyWXzzl08bQ3m3ygd3OQS_iUidrlmR3sFuHf2EAxYTyjFsK8IY7nSAuTtb32jXZVbybCD1oag](https://lh4.googleusercontent.com/UuBU2c3HKveL_42znHMRO-hDkK3qP1C1efMFnNmsj-tkM407_KYPM4SprhrEXg_9OQgkYGh6LgzzasRq8aoRkapePqQTO00OIIQnMBgn8hBtcZo7LMXP6rlyOxKXPEILGlyWXzzl08bQ3m3ygd3OQS_iUidrlmR3sFuHf2EAxYTyjFsK8IY7nSAuTtb32jXZVbybCD1oag)
    
- Değişkenler
    
    Terraform içerisinde yazılan kodlara direkt kullanıcı adı ve parola yazmak kodun değiştirilebilirliğini etkilediği için değişken verileri ayrı bir dosyada tutarak çalışabiliriz. Bu dosya genel olarak variables.tf olarak adlandırılır ve terraform komutu çalıştırıldığında main ile birlikte okunarak içerisindeki değerler kullanılır. Örnek vermek gerekirse;
    
    ```bash
    variable "vcenter" {
        default = "10.34.74.210"
    }
    ```
    
    Yukarıda vcenter adında bir değişken atanmış ve varsayılan olarak bir değer verilmiştir. Ancak bu değer değiştirilebilir.
    
- Komutlar
    
    Yukarıda bazı örnekler içerisinde açıkladım ancak yukarıdakinden daha fazla komut bulunuyor. Bu kısımda diğer komutları da açıklayacağım.
    
    ### Init
    
    Yazılan kod ile ilgili gerek duyulan paket ve modülleri indirmeyi sağlar. Çalıştırılmış bir kod üzerinde güncelleme yapıldığında ve yeni eklenen modülleri eklemek gerekirse `init -upgrade` ile çalıştırılır.
    
    ### Plan
    
    Yazılan kod çalıştığında oluşacak olan obje ve yapının nasıl olacağına dair bir önizleme yapmamızı sağlar.
    
    Komut tamamlandığında `terraform.tfstate` isimli bir dosya oluşur.
    
    ### Apply
    
    Yazılan kodu çalıştırmamızı sağlar. `Variable.tf` dosyasında boş bırakılan değerler `apply` komutunun ilk yürütüldüğü an kullanıcıya sorularak o alanlar doldurulur ve `provisioner` kısmı dahil tüm aşamalar komut satırından izlenebilir.
    
    Komut tamamlandığında varolan `terraform.tfstate` dosyası güncellenir, yoksa da oluşturulur.
    
    ### Destroy
    
    Kod ile oluşturduğumuz obje ve yapıları yıkmamızı sağlar.
    
    ### Show
    
    Plan ve Apply komutlarından sonra oluşan `terraform.tfstate` dosyasının içeriğini düzgün bir şekilde okuyabilmemizi sağlayan komuttur. Yoksa bir çıktı dönmez.
    
    ### Graph
    
    Plan ve Apply komutlarından sonra oluşan `terraform.tfstate` dosyasının içeriğini görselleştirebileceğimiz bir hale getirmemizi sağlar. Asıl olarak kendisi yine text formatında bir çıktı vermektedir ancak Linux üzerindeki dot aracı kullanarak çevirme işlemi yapılabilir. Örnek olması açısından yukarıdaki klonlanan makinenin `terraform.tfstate` dosyasının nasıl görselleşeceğine bakalım.
    
    Bu işlem için ihtiyacımız olan araç dot ve bu araç Debian türevi işletim sistemlerinde aşağıdaki komut ile kurulabilir.
    
    ```bash
    $ sudo apt install graphviz
    ```
    
    Sonrasında `graph` komutunun çıktısına bakalım.
    
    ```bash
    $ terraform graph
    digraph {
            compound = "true"
            newrank = "true"
            subgraph "root" {
                    "[root] data.vsphere_datacenter.dc (expand)" [label = "data.vsphere_datacenter.dc", shape = "box"]
                    ...
                    "[root] vsphere_virtual_machine.cloned_virtual_machine (expand)" -> "[root] var.vsphere_virtual_machine_name"
            }
    }
    ```
    
    Çıktıya baktığınızda plan ve apply komutunu yürüttüğümüzden içerik olarak farklı olmadığını ancak yazım şekli olarak farklı olduğunu görebiliriz. Şimdi komutumuzu yürüterek görselleştirme işlemini yapalım.
    
    ```bash
    $ terraform graph | dot -Tsvg > test01-vm.svg
    ```
    
    [https://lh5.googleusercontent.com/vzeUpGCKBF8WHfkfERaHyZijqAUMUneEbJtGD1GxZjcGJaneI50wGMFmkW2dL3ouLhEuxBkh_h0zF-P6YffLNMr9erzuQ6PpHgclh9pzrBi17v9PTqJmDRankJWsYay5kfM8KgUSR0p4GkaZ1uoQS_xVq4A8HDdnvD7hx-6xmjCT21oX4X9xdCJCgUoVFIhSsZTyk7VyTw](https://lh5.googleusercontent.com/vzeUpGCKBF8WHfkfERaHyZijqAUMUneEbJtGD1GxZjcGJaneI50wGMFmkW2dL3ouLhEuxBkh_h0zF-P6YffLNMr9erzuQ6PpHgclh9pzrBi17v9PTqJmDRankJWsYay5kfM8KgUSR0p4GkaZ1uoQS_xVq4A8HDdnvD7hx-6xmjCT21oX4X9xdCJCgUoVFIhSsZTyk7VyTw)
    
    Sonuç olarak yukarıdaki gibi bir svg dosyasına ulaşacağız. Bu dosyayı incelediğimizde klonlanan sanal makinenin verilerinin nerelerden geldiği ve hangi değişkenlerle işlemler yapıldığı görülebilir.
    
- VM Clone Örneği
    
    Yukarıda bahsettiklerimizi ve diğerlerini bir örnek üzerinden toparlamak gerekirse, elimizde bir ubuntu template sunucusu olduğunu ve bundan bir nginx server oluşturmak istediğimizi düşünelim. Bunun için öncelikle bir variable.tf oluşturacağız.
    
    ```bash
    $ cat variable.tf
    variable "vsphere_user" {
      default     = "sergen@vsphere.local"
    }
    variable "vsphere_password" {
        default = "powerfullpassword"
    }
    variable "vsphere_server" {
        default = "10.34.74.210"
    }
    variable "vsphere_datacenter" {
        default = "KKB01"
    }
    variable "vsphere_resource_pool" {
        default = "terraform-test"
    }
    variable "vsphere_datastore" {
        default = "LOCALDS1"
    }
    variable "vsphere_network" {
        default = "VM Network"
    }
    variable "vsphere_virtual_machine_template" { }
    variable "vsphere_virtual_machine_name" { }
    variable "VMubuntu_user"{
        default = "sergen"
    }
    variable "VMubuntu_password"{
        default = "lesspowerpassword"
    }
    ```
    
    Burada bazı tanımların içinin boş bırakıldığı görülebilir. Bunun sebebi burada kulanılacak olan değerleri kullanıcıdan almak istememizdir. Eğer kullanıcıdan alınması istenilen başka değerler varsa default tanım yapılması bırakılarak direkt kullanıcı tanımları ile işlem yapılabilir. Sırada main.tf dosyamız var.
    
    ```bash
    $ cat main.tf
    terraform {
      required_providers {
        vsphere = {
          source = "hashicorp/vsphere"
          version = "2.1.1"
        }
      }
    }
    
    provider "vsphere" {
      user           = "${var.vsphere_user}"
      password       = "${var.vsphere_password}"
      vsphere_server = "${var.vsphere_server}"
      allow_unverified_ssl = true
    }
    
    data "vsphere_datacenter" "dc" {
      name = "${var.vsphere_datacenter}"
    }
    data "vsphere_datastore" "datastore" {
      name          = "${var.vsphere_datastore}"
      datacenter_id = "${data.vsphere_datacenter.dc.id}"
    }
    data "vsphere_resource_pool" "pool" {
      name          = "${var.vsphere_resource_pool}"
      datacenter_id = "${data.vsphere_datacenter.dc.id}"
    }
    data "vsphere_network" "network" {
      name          = "${var.vsphere_network}"
      datacenter_id = "${data.vsphere_datacenter.dc.id}"
    }
    data "vsphere_virtual_machine" "template" {
      name          = "${var.vsphere_virtual_machine_template}"
      datacenter_id = "${data.vsphere_datacenter.dc.id}"
    }
    
    resource "vsphere_virtual_machine" "cloned_virtual_machine" {
      name             = "${var.vsphere_virtual_machine_name}"
      resource_pool_id = "${data.vsphere_resource_pool.pool.id}"
      datastore_id     = "${data.vsphere_datastore.datastore.id}"
    
      num_cpus = "${data.vsphere_virtual_machine.template.num_cpus}"
      memory   = "${data.vsphere_virtual_machine.template.memory}"
      guest_id = "${data.vsphere_virtual_machine.template.guest_id}"
    
      scsi_type = "${data.vsphere_virtual_machine.template.scsi_type}"
      network_interface {
        network_id   = "${data.vsphere_network.network.id}"
        adapter_type = "${data.vsphere_virtual_machine.template.network_interface_types[0]}"
      }
      disk {
        label = "disk0"
        size = "${data.vsphere_virtual_machine.template.disks.0.size}"
      }
      clone {
        template_uuid = "${data.vsphere_virtual_machine.template.id}"
      }
    
    provisioner "remote-exec" {
         connection {
            type     = "ssh"
            user     = var.VMubuntu_user
            password = var.VMubuntu_password
            host     = "${vsphere_virtual_machine.cloned_virtual_machine.default_ip_address}"
          }
        inline = [
          "echo ${var.password} | sudo hostnamectl set-hostname ${var.vmnew_hostname} “,
          “sudo -S apt-get -y update",
          "sudo apt-get -y install nginx",
          "sudo service nginx start",
        ]
      }
    }
    
    output "default_ip_address" {
     value = "${vsphere_virtual_machine.cloned_virtual_machine. default_ip_address}"
    }
    ```
    
    Şimdi burada çok fazla satır var ancak her biri okunduğunda anlaşılacak şeyler, örnek vermek gerekirse provider kısmından sonra gelen ilk data kısmı `vsphere_datacenter` olarak dc adıyla  daha önce `variables.tf` dosyasında tanımladığımız `vsphere_datacenter` değerini alacak. Benzer şekilde diğer kaynaklarda onayları ve makinenin kurulacağı yeri ayarlamak için aynı işlemi gerçekleştirecek şekilde konumlandırıldı. 
    Klonlama işlemini ise `vsphere_virtual_machine` kaynağının içerisindeki tanımlara göre yapılacaktır. Yine buradaki tanımlar yukarıda ve `variables.tf` dosyasından gelen değişkenlere göre yapılandırılacak ve klonlama işlemi sonrasında provisioner kısmındaki hostname ve paket komutları yürüyecektir. 
    Son olarak ise output olarak `terraform apply` komutunun çalıştırıldığı sunucuda sunucunun yeni hostname bilgisini görmemiz gerekmektedir. 
    Şimdi bu dosyaları çalışır hale getirmek için gerekli modülleri `initialize` etmesini sağlayalım.
    
    ```bash
    $ terraform init
    
    Initializing the backend...
    
    Initializing provider plugins...
    - Reusing previous version of hashicorp/vsphere from the dependency lock file
    - Using previously-installed hashicorp/vsphere v2.1.1
    
    Terraform has made some changes to the provider dependency selections recorded
    in the .terraform.lock.hcl file. Review those changes and commit them to your
    version control system if they represent changes you intended to make.
    
    Terraform has been successfully initialized!
    
    You may now begin working with Terraform. Try running "terraform plan" to see
    any changes that are required for your infrastructure. All Terraform commands
    should now work.
    
    If you ever set or change modules or backend configuration for Terraform,
    rerun this command to reinitialize your working directory. If you forget, other
    commands will detect it and remind you to do so if necessary.
    ```
    
    Sonrasında ise nasıl bir işlem yapacağına dair fikir edinmek için plan komutunu yürütelim.
    
    ```bash
    $ terraform plan
    var.vmnew_hostname
      Enter a value: nginx01
    var.vsphere_virtual_machine_name
      Enter a value: test01
    var.vsphere_virtual_machine_template
      Enter a value: VMubuntu
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    Terraform will perform the following actions:
      # vsphere_virtual_machine.cloned_virtual_machine will be created
      + resource "vsphere_virtual_machine" "cloned_virtual_machine" {
          + annotation                              = (known after apply)
          + boot_retry_delay                        = 10000
          + change_version                          = (known after apply)
          + cpu_limit                               = -1
          + cpu_share_count                         = (known after apply)
          + cpu_share_level                         = "normal"
          + datastore_id                            = "datastore-20"
          + default_ip_address                      = (known after apply)
          + ept_rvi_mode                            = "automatic"
          + firmware                                = "bios"
          + force_power_off                         = true
          + guest_id                                = "ubuntu64Guest"
          + guest_ip_addresses                      = (known after apply)
          + hardware_version                        = (known after apply)
          + host_system_id                          = (known after apply)
          + hv_mode                                 = "hvAuto"
          + id                                      = (known after apply)
          + ide_controller_count                    = 2
          + imported                                = (known after apply)
          + latency_sensitivity                     = "normal"
          + memory                                  = 4096
          + memory_limit                            = -1
          + memory_share_count                      = (known after apply)
          + memory_share_level                      = "normal"
          + migrate_wait_timeout                    = 30
          + moid                                    = (known after apply)
          + name                                    = "test01"
          + num_cores_per_socket                    = 1
          + num_cpus                                = 2
          + power_state                             = (known after apply)
          + poweron_timeout                         = 300
          + reboot_required                         = (known after apply)
          + resource_pool_id                        = "resgroup-183"
          + run_tools_scripts_after_power_on        = true
          + run_tools_scripts_after_resume          = true
          + run_tools_scripts_before_guest_shutdown = true
          + run_tools_scripts_before_guest_standby  = true
          + sata_controller_count                   = 0
          + scsi_bus_sharing                        = "noSharing"
          + scsi_controller_count                   = 1
          + scsi_type                               = "lsilogic"
          + shutdown_wait_timeout                   = 3
          + storage_policy_id                       = (known after apply)
          + swap_placement_policy                   = "inherit"
          + tools_upgrade_policy                    = "manual"
          + uuid                                    = (known after apply)
          + vapp_transport                          = (known after apply)
          + vmware_tools_status                     = (known after apply)
          + vmx_path                                = (known after apply)
          + wait_for_guest_ip_timeout               = 0
          + wait_for_guest_net_routable             = true
          + wait_for_guest_net_timeout              = 5
    
          + clone {
              + template_uuid = "421b22c2-ae17-cc67-af09-cc5756bb4f7f"
              + timeout       = 30
            }
    
          + disk {
              + attach            = false
              + controller_type   = "scsi"
              + datastore_id      = "<computed>"
              + device_address    = (known after apply)
              + disk_mode         = "persistent"
              + disk_sharing      = "sharingNone"
              + eagerly_scrub     = false
              + io_limit          = -1
              + io_reservation    = 0
              + io_share_count    = 0
              + io_share_level    = "normal"
              + keep_on_remove    = false
              + key               = 0
              + label             = "disk0"
              + path              = (known after apply)
              + size              = 20
              + storage_policy_id = (known after apply)
              + thin_provisioned  = true
              + unit_number       = 0
              + uuid              = (known after apply)
              + write_through     = false
            }
    
          + network_interface {
              + adapter_type          = "vmxnet3"
              + bandwidth_limit       = -1
              + bandwidth_reservation = 0
              + bandwidth_share_count = (known after apply)
              + bandwidth_share_level = "normal"
              + device_address        = (known after apply)
              + key                   = (known after apply)
              + mac_address           = (known after apply)
              + network_id            = "network-25"
            }
        }
    
    Plan: 1 to add, 0 to change, 0 to destroy.
    
    Changes to Outputs:
      + default_ip_address = (known after apply)
    
    ──────────────────────────────────────────────────────────────────────────────────────
    Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
    ```
    
    Çıktıyı okuduğunuzda ilk olarak kalın şekilde belirtilmiş satırlar görüyoruz. Bunlar `variables.tf` dosyasında `default` tanım verilmeyen değişkenlerdir. Sonrasında kod olarak yazdığımızdan daha fazla seçenek olduğunu görebilirsiniz. Bu aslında kod içerisinde yazmadığımızda varsayılan olarak gelen bütün argümanları içermektedir. Bazı satırlarda apply sonrasında açıklamasını görüyoruz. Bunun sebebi elimizde oluşmuş bir makine olmadığı için o kısımların alacağı değerler olmamasıdır.
    
    Komutu yürüttüğümüzde ise aşağıdaki çıktıya ulaşacağız.
    
    ```bash
    $ time terraform apply
    var.vmnew_hostname
      Enter a value: nginx01
    var.vsphere_virtual_machine_name
      Enter a value: test01
    var.vsphere_virtual_machine_template
      Enter a value: VMubuntu
    
    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create
    Terraform will perform the following actions:
      # vsphere_virtual_machine.cloned_virtual_machine will be created
      + resource "vsphere_virtual_machine" "cloned_virtual_machine" {
    ...
    
    Plan: 1 to add, 0 to change, 0 to destroy.
    
    Changes to Outputs:
      + my_hostname = (known after apply)
    
    Do you want to perform these actions?
      Terraform will perform the actions described above.
      Only 'yes' will be accepted to approve.
    
      Enter a value: yes
    
    vsphere_virtual_machine.cloned_virtual_machine: Creating...
    vsphere_virtual_machine.cloned_virtual_machine: Still creating... [10s elapsed]
    ...
    vsphere_virtual_machine.cloned_virtual_machine: Still creating... [1m40s elapsed]
    vsphere_virtual_machine.cloned_virtual_machine: Provisioning with 'remote-exec'...
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec): Connecting to remote host via SSH...
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Host: 10.34.74.227
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   User: sergen
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Password: true
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Private key: false
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Certificate: false
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   SSH Agent: false
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Checking Host Key: false
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec):   Target Platform: unix
    vsphere_virtual_machine.cloned_virtual_machine (remote-exec): Connected!
    ...
    vsphere_virtual_machine.cloned_virtual_machine: Creation complete after 2m7s [id=421bd709-0151-4991-a023-32a61e924a4c]
    
    Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
    
    Outputs:
    
    default_ip_address = "10.34.74.227"
    
    real    2m42.140s
    user    0m2.220s
    sys     0m0.842s
    ```
    

- Kaynaklar
    
    [https://www.techbeatly.com/terraform-cheat-sheet/](https://www.techbeatly.com/terraform-cheat-sheet/)
    
    [https://upcloud.com/community/tutorials/terraform-variables/](https://upcloud.com/community/tutorials/terraform-variables/)
    
    [https://medium.com/spacelift/terraform-best-practices-for-better-infrastructure-management-49e0859b5537](https://medium.com/spacelift/terraform-best-practices-for-better-infrastructure-management-49e0859b5537)
    
    [https://www.altaro.com/vmware/secure-terraform-deployment/](https://www.altaro.com/vmware/secure-terraform-deployment/)
    
    [https://emrecicek.net/blog/terraform-ve-iac/](https://emrecicek.net/blog/terraform-ve-iac/)
    
    - Kurs
        
        [https://www.udemy.com/course/terraform-cloud/?referralCode=538B23A2456C61D64A47](https://www.udemy.com/course/terraform-cloud/?referralCode=538B23A2456C61D64A47)
        
    - AWS
        
        [https://guneycansanli.github.io/my-blog/terraform-installation-and-fundamentals-with-aws/](https://guneycansanli.github.io/my-blog/terraform-installation-and-fundamentals-with-aws/)
        
    - Azure
        
        [https://www.devopsschool.com/blog/terraform-example-code-for-create-azure-linux-windows-vm-with-file-remote-exec-local-exec-provisioner/](https://www.devopsschool.com/blog/terraform-example-code-for-create-azure-linux-windows-vm-with-file-remote-exec-local-exec-provisioner/)
        
    - vSphere
        
        [https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs#example-usage](https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs#example-usage)
        
        [https://www.virtualizationhowto.com/2021/05/terraform-vsphere-tutorial-linux-virtual-machine-clone/](https://www.virtualizationhowto.com/2021/05/terraform-vsphere-tutorial-linux-virtual-machine-clone/)
        
        [https://www.youtube.com/watch?v=3rE2ohOV_5U](https://www.youtube.com/watch?v=3rE2ohOV_5U)
        
        [https://sdorsett.github.io/post/2018-12-24-using-terraform-to-clone-a-virtual-machine-on-vsphere/](https://sdorsett.github.io/post/2018-12-24-using-terraform-to-clone-a-virtual-machine-on-vsphere/)
        
        [https://sdorsett.github.io/post/2018-12-26-using-local-exec-and-remote-exec-provisioners-with-terraform/](https://sdorsett.github.io/post/2018-12-26-using-local-exec-and-remote-exec-provisioners-with-terraform/)
        
        [https://buildvirtual.net/building-vsphere-virtual-machines-with-terraform/](https://buildvirtual.net/building-vsphere-virtual-machines-with-terraform/)
        
        [https://github.com/Terraform-VMWare-Modules/terraform-vsphere-vm](https://github.com/Terraform-VMWare-Modules/terraform-vsphere-vm)
        
        [https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs/data-sources/virtual_machine#alternate_guest_name](https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs/data-sources/virtual_machine#alternate_guest_name)
        
        [https://buildvirtual.net/building-vsphere-virtual-machines-with-terraform/](https://buildvirtual.net/building-vsphere-virtual-machines-with-terraform/)
        
        [https://github.com/linoproject/terraform/tree/master/vsphere-cloudinit](https://github.com/linoproject/terraform/tree/master/vsphere-cloudinit)
        
    - Troubleshoot
        
        [https://stackoverflow.com/questions/59640285/how-to-introduce-reboot-option-in-remote-exec-terraform](https://stackoverflow.com/questions/59640285/how-to-introduce-reboot-option-in-remote-exec-terraform)
        
        [https://github.com/hashicorp/terraform-provider-vsphere/issues/1202](https://github.com/hashicorp/terraform-provider-vsphere/issues/1202)
        
        [https://stackoverflow.com/questions/67272588/configure-provisioning-script-for-multiple-instance-vm-vsphere-deployment](https://stackoverflow.com/questions/67272588/configure-provisioning-script-for-multiple-instance-vm-vsphere-deployment)
        
        [https://anthonyspiteri.net/quick-fix-terraform-linux-remote-exec-provisioner-errors-without-exit-status/](https://anthonyspiteri.net/quick-fix-terraform-linux-remote-exec-provisioner-errors-without-exit-status/)
        
        [https://stackoverflow.com/questions/37847273/how-to-run-sudo-commands-in-terraform](https://stackoverflow.com/questions/37847273/how-to-run-sudo-commands-in-terraform)
        

- Notion AI Çıktısı
    
    Bu Terraform kodu, Azure üzerinde sanal makine oluşturmanızı sağlar.
    
    ```vhdl
    # Azure Provider Konfigurasyonu
    provider "azurerm" {
      features {}
    }
    
    # Azure Sanal Makine Oluşturma
    resource "azurerm_virtual_machine" "example" {
      name                  = "example-vm"
      location              = "West Europe"
      resource_group_name   = "example-resources"
      network_interface_ids = [azurerm_network_interface.example.id]
      vm_size               = "Standard_DS1_v2"
    
      storage_image_reference {
        publisher = "Canonical"
        offer     = "UbuntuServer"
        sku       = "18.04-LTS"
        version   = "latest"
      }
    
      storage_os_disk {
        name              = "example-osdisk"
        caching           = "ReadWrite"
        create_option     = "FromImage"
        managed_disk_type = "Standard_LRS"
      }
    
      os_profile {
        computer_name  = "examplevm"
        admin_username = "exampleuser"
        admin_password = "P@ssw0rd1234!"
      }
    
      os_profile_linux_config {
        disable_password_authentication = false
      }
    }
    
    # Azure Sanal Makine Ağ Arayüzü
    resource "azurerm_network_interface" "example" {
      name                = "example-nic"
      location            = "West Europe"
      resource_group_name = "example-resources"
    
      ip_configuration {
        name                          = "internal"
        subnet_id                     = azurerm_subnet.example.id
        private_ip_address_allocation = "Dynamic"
      }
    }
    
    # Azure Sanal Makine Alt Ağı
    resource "azurerm_subnet" "example" {
      name                 = "example-subnet"
      resource_group_name  = "example-resources"
      virtual_network_name = azurerm_virtual_network.example.name
      address_prefixes     = ["10.0.2.0/24"]
    }
    
    # Azure Sanal Makine Sanal Ağ
    resource "azurerm_virtual_network" "example" {
      name                = "example-network"
      address_space       = ["10.0.0.0/16"]
      location            = "West Europe"
      resource_group_name = "example-resources"
    }
    
    ```
    
    Bu kodu kullanarak, Azure'da `example-vm` adında yeni bir sanal makine oluşturabilirsiniz. Unutmayın, kodu kendi ihtiyaçlarınıza göre uyarlamalısınız.
    
    Sanal makine oluşturma kodunun yanı sıra, ayrıca Terraform ile oluşturacağınız Azure kaynaklarının kimlik doğrulaması için kullanılan Azure kimlik bilgilerini de tanımlamanız gerekmektedir. Bu kimlik bilgileri Azure hesabınızdan elde edilebilir.
    
    Aşağıdaki örnekte, Azure kimlik bilgileri `azurerm` provider'ına verilmiştir. Bu örnekte, Azure hesap kimlik bilgileri ortak kullanılan bir varsayılan dosyada tutulmaktadır. Bunun yerine, Azure kimlik bilgilerinizi doğrudan kodda belirtebilirsiniz.
    
    ```
    # Azure Provider Kimlik Doğrulaması
    provider "azurerm" {
      features {}
    
      subscription_id = var.subscription_id
      client_id       = var.client_id
      client_secret   = var.client_secret
      tenant_id       = var.tenant_id
    }
    
    ```
    
    Yukarıdaki kodda, `subscription_id`, `client_id`, `client_secret` ve `tenant_id` gibi değişkenler, Azure kimlik bilgilerini depolamak için kullanılır.
    
    Terraform ile Azure sanal makinesi oluşturma konusunda daha fazla bilgi edinmek isterseniz, [Terraform Azure Sanal Makine](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_machine) ve [Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) belgelerine göz atabilirsiniz.