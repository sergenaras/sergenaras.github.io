# MSSQL with Pacemaker

Created by: Sergen
Created time: December 18, 2022 12:23 PM
Last edited by: Sergen
Last edited time: April 17, 2023 10:49 AM
Status: In progress
Tags: veritabanı
Type: Research

Elimizde üç adet sunucu var bunların üzerinde SQL Cluster kurulumu yapacağız.

| Hostname | IP | OS | Kernel |
| --- | --- | --- | --- |
| pace221 | 10.34.74.221 | Ubuntu 20.04.5 LTS (Focal Fossa) | 5.4.0-117-generic |
| pace222 | 10.34.74.222 | Ubuntu 20.04.5 LTS (Focal Fossa) | 5.4.0-117-generic |
| pace223 | 10.34.74.223 | Ubuntu 20.04.5 LTS (Focal Fossa) | 5.4.0-117-generic |
- **Timezone**
    
    Bu tarz eş zamanlı çalışması gereken sunucular için saat ayarlarının eşit olması önemlidir. Bunun için öncelikle aşağıdaki script’i çalıştırıyoruz.
    
    ```bash
    curl -so ~/timezone.sh https://raw.githubusercontent.com/sergenaras/bls/main/timezone.sh && bash ~/timezone.sh
    ```
    
- **MS SQL Server Kurulumu**
    
    ```bash
    sudo wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
    sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-preview.list)"
    sudo apt-get update
    sudo apt-get install -y mssql-server
    ```
    
    Paket deposunu ekleyip MSSQL kurulumu gerçekleştirdikten sonra konfigürasyon ayarlarını yapmamız gerekecek..
    
    ```bash
    sudo /opt/mssql/bin/mssql-conf setup
    ```
    
    Son olarak servisin durumunu kontrol edelim.
    
- **MS SQL Client Kurulumu**
    
    Komut satırı üzerinden MSSQL’e bağlantı sağlayabilmek için kurulum yapalım. Öncelikle paket ve paket anahtarlarını indirmek için gerekli paketleri indirelim.
    
    ```bash
    sudo apt-get update 
    sudo apt install curl
    ```
    
    Şimdi paket ve kaynak depo dosyalarını indirebiliriz.
    
    ```bash
    curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
    curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list | sudo tee /etc/apt/sources.list.d/msprod.list
    ```
    
    Paket ve anahtar depoları hazır olduğu için gerekli araçları indirebiliriz.
    
    ```bash
    sudo apt-get update 
    sudo apt-get install mssql-tools unixodbc-dev
    ```
    
    Son olarak komutları yürütebilmek için path ayarlarını yapalım.
    
    ```bash
    echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bash_profile
    echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
    source ~/.bashrc
    ```
    
    Kurulum sonrasında bağlantı denemek için aşağıdaki komutu yürütebiliriz.
    
    ```bash
    sqlcmd -S localhost -U sa -P
    ```
    
- **MS SQL Always On Altyapısı**
    
    Ayrı ayrı kurduğumuz üç MSSQL’i bir cluster haline getirmek için öncelikle dns ayarlarını yapmamız gerekmektedir. Bunun için mevcut sunucuların baktığı bir dns sunucumuz olmadığı için sunucularımızın `/etc/hosts` dosyasına aşağıdaki dns kayıtlarını ekliyoruz.
    
    ```bash
    sudo cat /etc/hosts
    10.34.74.221 pace221
    10.34.74.222 pace222
    10.34.74.223 pace223
    ```
    
    Always On ile ilgili ayarı aktif hale getiriyoruz.
    
    ```bash
    sudo /opt/mssql/bin/mssql-conf set hadr.hadrenabled 1
    sudo systemctl restart mssql-server
    ```
    
    Şimdi MSSQL üzerinden ayarları yapmaya başlayabiliriz. Öncelikle her üç intance’da da olası sorunları tespit edebilmek adına `AlwaysOn_health` event’ini aktif hale getiriyoruz.
    
    ```sql
    ALTER EVENT SESSION AlwaysOn_health ON SERVER WITH (STARTUP_STATE=ON);
    GO
    ```
    
    İletişimi güvenli hale getirmek için ana sunucumuzda sertifika oluşturuyoruz. Bu da şuan **pace221** üzerinde şu SQL komutlarını çalıştıracağımız anlamına geliyor.
    
    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterPassw0rd!';
    CREATE CERTIFICATE dbm_certificate WITH SUBJECT = 'dbm';
    BACKUP CERTIFICATE dbm_certificate
       TO FILE = '/var/opt/mssql/data/dbm_certificate.cer'
       WITH PRIVATE KEY (
               FILE = '/var/opt/mssql/data/dbm_certificate.pvk',
               ENCRYPTION BY PASSWORD = 'PrivatePassw0rd!'
            );
    ```
    
    Komutları yürüttükten sonra `/var/opt/mssql/data/` dizini içerisinde sertifika ve anahtarlarımız oluştu.
    
    ```bash
    @pace221
    ls -ltr /var/opt/mssql/data/
    -rw-rw---- 1 mssql mssql     1788 Aug 26 10:34 dbm_certificate.pvk
    -rw-rw---- 1 mssql mssql      923 Aug 26 10:34 dbm_certificate.cer
    ```
    
    Bu dosyaları diğer sunucularımıza aktarmak için arşive alalım.
    
    ```bash
    sudo tar -cvf /home/sergen/mssql.tar /var/opt/mssql/data/dbm_certificate*
    ```
    
    Bu dosyaları diğer sunuculara taşıyalım
    
    ```bash
    scp /home/sergen/mssql.tar sergen@pace222:/home/sergen/
    scp /home/sergen/mssql.tar sergen@pace223:/home/sergen/
    ```
    
    Dışarı çıkartıp `/var/opt/mssql/data` lokasyonuna taşıyıp `chown` ile aitlik durumunu değiştirelim.
    
    ```bash
    @pace222
    tar -xvf mssql.tar 
    dbm_certificate.cer
    dbm_certificate.pvk
    
    mv dbm* /var/opt/mssql/data/
    chown mssql:mssql /var/opt/mssql/data/dbm_certificate.*
    
    ls -ltr /var/opt/mssql/data/dbm_certificate.*
    -rw-r----- 1 mssql mssql 1788 Dec 18 14:05 /var/opt/mssql/data/dbm_certificate.pvk
    -rw-r----- 1 mssql mssql  923 Dec 18 14:05 /var/opt/mssql/data/dbm_certificate.cer
    ```
    
    ```bash
    @pace223
    tar -xvf mssql.tar 
    dbm_certificate.cer
    dbm_certificate.pvk
    
    sudo mv dbm* /var/opt/mssql/data/
    sudo chown mssql:mssql /var/opt/mssql/data/dbm_certificate.*
    
    sudo ls -ltr /var/opt/mssql/data/dbm_certificate.*
    -rw-r----- 1 mssql mssql 1788 Dec 18 14:05 /var/opt/mssql/data/dbm_certificate.pvk
    -rw-r----- 1 mssql mssql  923 Dec 18 14:05 /var/opt/mssql/data/dbm_certificate.cer
    ```
    
    Bu kopyalama işlemlerinden sonra ikincil sunucularımızda bu sertifikaların kabulu için aşağıdaki SQL komutlarını yürütmeliyiz.
    
    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'MasterPassw0rd!';
    CREATE CERTIFICATE dbm_certificate
        FROM FILE = '/var/opt/mssql/data/dbm_certificate.cer'
        WITH PRIVATE KEY (
               FILE = '/var/opt/mssql/data/dbm_certificate.pvk',
               DECRYPTION BY PASSWORD = 'PrivatePassw0rd!'
            );
    ```
    
    Bu işlemleri de tamamladıktan sonra temel sertifika işlemlerini halletmiş olduk. Sırada veri tabanlarının birbirlerine eşitleneceği endpoint’leri oluşturmak var. Bunu oluştururken yukarıdaki sertifikalardan da faydalanıyoruz.
    
    ```sql
    CREATE ENDPOINT [Hadr_endpoint]
        AS TCP (LISTENER_PORT = 5022)
        FOR DATABASE_MIRRORING (
            ROLE = ALL,
            AUTHENTICATION = CERTIFICATE dbm_certificate,
            ENCRYPTION = REQUIRED ALGORITHM AES
            );
    
    ALTER ENDPOINT [Hadr_endpoint] STATE = STARTED;
    ```
    
    Yukarıdaki SQL komutlarını tüm sunucularımızda yürüttükten sonra Always On için gerekli olan altyapıyı sağlamış olduk. Tüm sunucularımızda aşağıdaki çıktıyı görmemiz gerekiyor.
    
    ```bash
    ss -tulpn | grep 5022
    tcp     LISTEN   0        128              0.0.0.0:5022          0.0.0.0:*       users:(("sqlservr",pid=272555,fd=64))                                          
    tcp     LISTEN   0        128                    *:5022                *:*       users:(("sqlservr",pid=272555,fd=127))
    ```
    
    Bunu gördüysek sırada Availability Group oluşturma kısmına geçebiliriz.
    
- **MS SQL - Pacemaker Kullanıcısı**
    
    Availability Group kurabilmek ve sonrasında pacemaker üzerinden kontrol sağlayabilmek adına bir ortak kullanıcı gerekmektedir. Bundan ötürü tüm sunucularda kullanıcı oluşturmamız gerekmektedir.
    
    ```sql
    USE [master]
    GO
    CREATE LOGIN [pacemakerLogin] with PASSWORD= N'KompleksPassw0rd!';
    
    ALTER SERVER ROLE [sysadmin] ADD MEMBER [pacemakerLogin];
    ```
    
    Sonrasında ihtiyaç halinde okuyabilmesi için mssql dizini içerisine aşağıdaki dosyaya yazalım. 
    
    ```bash
    echo 'pacemakerLogin' >> ~/pacemaker-passwd
    echo 'KompleksPassw0rd!' >> ~/pacemaker-passwd
    sudo mv ~/pacemaker-passwd /var/opt/mssql/secrets/passwd
    sudo chown root:root /var/opt/mssql/secrets/passwd
    sudo chmod 400 /var/opt/mssql/secrets/passwd
    ```
    
- **MS SQL Availability Group - Master**
    
    Birden fazla Availability Group oluşturma seçeneğimiz var ancak şuan elimizdeki her şeyi kullanarak oluşturduğumuz seçenek üzerinden gideceğiz. Detaylar için [Microsoft](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-configure-ha?view=sql-server-ver16) dokümantasyonu incelenebilir.
    
    Master sunucumuzda aşağıdaki SQL komutunu yürütelim.
    
    ```sql
    CREATE AVAILABILITY GROUP [ag0]
         WITH (DB_FAILOVER = ON, CLUSTER_TYPE = EXTERNAL)
         FOR REPLICA ON
             N'pace221' 
     	      	WITH (
      	       ENDPOINT_URL = N'tcp://pace221:5022',
      	       AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
      	       FAILOVER_MODE = EXTERNAL,
      	       SEEDING_MODE = AUTOMATIC
      	       ),
             N'pace222' 
      	    WITH ( 
      	       ENDPOINT_URL = N'tcp://pace222:5022', 
      	       AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
      	       FAILOVER_MODE = EXTERNAL,
      	       SEEDING_MODE = AUTOMATIC
      	       ),
      	   N'pace223'
             WITH( 
      	      ENDPOINT_URL = N'tcp://pace223:5022', 
      	      AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
      	      FAILOVER_MODE = EXTERNAL,
      	      SEEDING_MODE = AUTOMATIC
      	      );
    
    ALTER AVAILABILITY GROUP [ag0] GRANT CREATE ANY DATABASE;
    ```
    
    Bu oluşturduğumuz grub üzerinde daha önce oluşturduğumuz kullanıcının yetkilendirmesini yapalım.
    
    ```sql
    GRANT ALTER, CONTROL, VIEW DEFINITION ON AVAILABILITY GROUP::ag0 TO pacemakerLogin
    GRANT VIEW SERVER STATE TO pacemakerLogin
    ```
    
- **MS SQL Availability Group - Secondary**
    
    İkincil sunucuların bağlantısı için aşağıdaki komutları yürütebiliriz.
    
    ```sql
    ALTER AVAILABILITY GROUP [ag0] JOIN WITH (CLUSTER_TYPE = EXTERNAL);
    ALTER AVAILABILITY GROUP [ag0] GRANT CREATE ANY DATABASE;
    ```
    
    Bu işlem diğerlerine nazaran biraz daha uzun sürebilir ancak sonuç olarak eğer bu sunucu master aşamasındaki SQL komutu içerisinde varsa sorun çıkmadan ekleyecektir. 
    
- **MS SQL Availability Group - Database**
    
    Oluşturulan Availability Group tarafından erişilebilen veritabanları için alter ile erişilebilirlik verilmelidir.
    
    Bu veritabanı sadece **master** üzerinde oluşturulabilir.
    
    ```sql
    CREATE DATABASE [db1];
    ALTER DATABASE [db1] SET RECOVERY FULL;
    BACKUP DATABASE [db1] 
       TO DISK = N'/var/opt/mssql/data/db1.bak';
    
    ALTER AVAILABILITY GROUP [ag0] ADD DATABASE [db1];
    ```
    
- **MS SQL Availability Group - Test**
    
    Buraya kadar tüm işlemler başarı ile sonuçlandı ise aşağıdaki SQL komutunun her üç sunucuda da aynı sonucu dönmesi gerekiyor.
    
    ```sql
    SELECT * FROM sys.databases WHERE name = 'db1';
    GO
    SELECT DB_NAME(database_id) AS 'database', synchronization_state_desc FROM sys.dm_hadr_database_replica_states;
    ```
    
    ![Master Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_15.43.39.png)
    
    Master Server
    
    Master sunucu üzerinden bakıldığımda yukarıdaki gibi bütün veritabanları ve onların senkron durumunu görebiliriz. 
    
    ![Secondary Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_15.44.56.png)
    
    Secondary Server
    
    Secondary sunucularda ise sadece hangi veritabanlarının senkron olduğu durumunu gösteriyor. 
    
- **MS SQL Availability Group - Master Check**
    
    İlgili instance üzerinde bulunan tüm availability group’ları görmek için aşağıdaki SQL komutunu yürütebiliriz.
    
    ```sql
    SELECT Groups.[Name] AS AGname
    FROM sys.dm_hadr_availability_group_states States
    INNER JOIN master.sys.availability_groups Groups ON States.group_id = Groups.group_id
    WHERE primary_replica = @@Servername;
    ```
    
    ![Master Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.34.58.png)
    
    Master Server
    
    Üzerinde Availability Group olan MS SQL Instance’da komut çalıştırıldığında yukarıdaki sonucu elde edeceğiz.
    
    ![Secondary Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.34.45.png)
    
    Secondary Server
    
    Üzerinde Availability Group oluşturulmamış MS SQL Instance’da ise çıktı yukarıdaki gibi olacaktır. 
    
- **MS SQL Availability Group - Secondary Check**
    
    İkincil sunucular üzerinden bağlı olunan availability group ve veritabanlarını görmek için aşağıdaki SQL komutu yürütülebilir.
    
    ```sql
    SELECT
    Groups.[Name] AS AGname,
    AGDatabases.database_name AS Databasename
    FROM sys.dm_hadr_availability_group_states States
    INNER JOIN master.sys.availability_groups Groups ON States.group_id = Groups.group_id
    INNER JOIN sys.availability_databases_cluster AGDatabases ON Groups.group_id = AGDatabases.group_id
    WHERE primary_replica != @@Servername
    ORDER BY
    AGname ASC,
    Databasename ASC;
    ```
    
    ![Master Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.37.40.png)
    
    Master Server
    
    Bu MS SQL Instance içerisinde dışarıdan bağlanmış bir Availability Group olmadığı için çıktı boş dönmektedir.
    
    ![Secondary Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.37.50.png)
    
    Secondary Server
    
    Bu MS SQL Instance üzerinde ise kurduğumuz ilk Availability Group ve onun üzerinde çalıştırdığımız veritabanı olduğu için bunu görmekteyiz.
    
- **MS SQL Availability Group - General Check**
    
    Bir Availability Group’a bağlı olan sunucular, veritabanları, rolleri, failover durumları gibi genel bilgileri görmek için aşağıdaki betik kullanılabilir.
    
    ```sql
    SET NOCOUNT ON;
    
    DECLARE @AGname NVARCHAR(128);
    
    DECLARE @SecondaryReplicasOnly BIT;
    
    SET @AGname = 'ag0';	    --SET AGname for a specific AG for SET to NULL for ALL AG's
    
    IF OBJECT_ID('TempDB..#tmpag_availability_groups') IS NOT NULL
    DROP TABLE [#tmpag_availability_groups];
    
    SELECT *
    INTO [#tmpag_availability_groups]
    FROM   [master].[sys].[availability_groups];
    
    IF(@AGname IS NULL
    OR EXISTS
    (
    SELECT [Name]
    FROM   [#tmpag_availability_groups]
    WHERE  [Name] = @AGname
    ))
    BEGIN
    
    IF OBJECT_ID('TempDB..#tmpdbr_availability_replicas') IS NOT NULL
    DROP TABLE [#tmpdbr_availability_replicas];
    
    IF OBJECT_ID('TempDB..#tmpdbr_database_replica_cluster_states') IS NOT NULL
    DROP TABLE [#tmpdbr_database_replica_cluster_states];
    
    IF OBJECT_ID('TempDB..#tmpdbr_database_replica_states') IS NOT NULL
    DROP TABLE [#tmpdbr_database_replica_states];
    
    IF OBJECT_ID('TempDB..#tmpdbr_database_replica_states_primary_LCT') IS NOT NULL
    DROP TABLE [#tmpdbr_database_replica_states_primary_LCT];
    
    IF OBJECT_ID('TempDB..#tmpdbr_availability_replica_states') IS NOT NULL
    DROP TABLE [#tmpdbr_availability_replica_states];
    
    SELECT [group_id],
    [replica_id],
    [replica_server_name],
    [availability_mode],
    [availability_mode_desc]
    INTO [#tmpdbr_availability_replicas]
    FROM   [master].[sys].[availability_replicas];
    
    SELECT [replica_id],
    [group_database_id],
    [database_name],
    [is_database_joined],
    [is_failover_ready]
    INTO [#tmpdbr_database_replica_cluster_states]
    FROM   [master].[sys].[dm_hadr_database_replica_cluster_states];
    
    SELECT *
    INTO [#tmpdbr_database_replica_states]
    FROM   [master].[sys].[dm_hadr_database_replica_states];
    
    SELECT [replica_id],
    [role],
    [role_desc],
    [is_local]
    INTO [#tmpdbr_availability_replica_states]
    FROM   [master].[sys].[dm_hadr_availability_replica_states];
    
    SELECT [ars].[role],
    [drs].[database_id],
    [drs].[replica_id],
    [drs].[last_commit_time]
    INTO [#tmpdbr_database_replica_states_primary_LCT]
    FROM   [#tmpdbr_database_replica_states] AS [drs]
    LEFT JOIN [#tmpdbr_availability_replica_states] [ars] ON [drs].[replica_id] = [ars].[replica_id]
    WHERE  [ars].[role] = 1;
    
    SELECT [AG].[name] AS [AvailabilityGroupName],
    [AR].[replica_server_name] AS [AvailabilityReplicaServerName],
    [dbcs].[database_name] AS [AvailabilityDatabaseName],
    ISNULL([dbcs].[is_failover_ready],0) AS [IsFailoverReady],
    ISNULL([arstates].[role_desc],3) AS [ReplicaRole],
    [AR].[availability_mode_desc] AS [AvailabilityMode],
    CASE [dbcs].[is_failover_ready]
    WHEN 1
    THEN 0
    ELSE ISNULL(DATEDIFF([ss],[dbr].[last_commit_time],[dbrp].[last_commit_time]),0)
    END AS [EstimatedDataLoss_(Seconds)],
    ISNULL(CASE [dbr].[redo_rate]
    WHEN 0
    THEN-1
    ELSE CAST([dbr].[redo_queue_size] AS FLOAT) / [dbr].[redo_rate]
    END,-1) AS [EstimatedRecoveryTime_(Seconds)],
    ISNULL([dbr].[is_suspended],0) AS [IsSuspended],
    ISNULL([dbr].[suspend_reason_desc],'-') AS [SuspendReason],
    ISNULL([dbr].[synchronization_state_desc],0) AS [SynchronizationState],
    ISNULL([dbr].[last_received_time],0) AS [LastReceivedTime],
    ISNULL([dbr].[last_redone_time],0) AS [LastRedoneTime],
    ISNULL([dbr].[last_sent_time],0) AS [LastSentTime],
    ISNULL([dbr].[log_send_queue_size],-1) AS [LogSendQueueSize],
    ISNULL([dbr].[log_send_rate],-1) AS [LogSendRate_KB/S],
    ISNULL([dbr].[redo_queue_size],-1) AS [RedoQueueSize_KB],
    ISNULL([dbr].[redo_rate],-1) AS [RedoRate_KB/S],
    ISNULL(CASE [dbr].[log_send_rate]
    WHEN 0
    THEN-1
    ELSE CAST([dbr].[log_send_queue_size] AS FLOAT) / [dbr].[log_send_rate]
    END,-1) AS [SynchronizationPerformance],
    ISNULL([dbr].[filestream_send_rate],-1) AS [FileStreamSendRate],
    ISNULL([dbcs].[is_database_joined],0) AS [IsJoined],
    [arstates].[is_local] AS [IsLocal],
    ISNULL([dbr].[last_commit_lsn],0) AS [LastCommitLSN],
    ISNULL([dbr].[last_commit_time],0) AS [LastCommitTime],
    ISNULL([dbr].[last_hardened_lsn],0) AS [LastHardenedLSN],
    ISNULL([dbr].[last_hardened_time],0) AS [LastHardenedTime],
    ISNULL([dbr].[last_received_lsn],0) AS [LastReceivedLSN],
    ISNULL([dbr].[last_redone_lsn],0) AS [LastRedoneLSN]
    FROM   [#tmpag_availability_groups] AS [AG]
    INNER JOIN [#tmpdbr_availability_replicas] AS [AR] ON [AR].[group_id] = [AG].[group_id]
    INNER JOIN [#tmpdbr_database_replica_cluster_states] AS [dbcs] ON [dbcs].[replica_id] = [AR].[replica_id]
    LEFT OUTER JOIN [#tmpdbr_database_replica_states] AS [dbr] ON [dbcs].[replica_id] = [dbr].[replica_id]
    AND [dbcs].[group_database_id] = [dbr].[group_database_id]
    LEFT OUTER JOIN [#tmpdbr_database_replica_states_primary_LCT] AS [dbrp] ON [dbr].[database_id] = [dbrp].[database_id]
    INNER JOIN [#tmpdbr_availability_replica_states] AS [arstates] ON [arstates].[replica_id] = [AR].[replica_id]
    WHERE  [AG].[name] = ISNULL(@AGname,[AG].[name])
    ORDER BY [AvailabilityReplicaServerName] ASC,
    [AvailabilityDatabaseName] ASC;
    
    /*********************/
    
    END;
    ELSE
    BEGIN
    RAISERROR('Invalid AG name supplied, please correct and try again',12,0);
    END;
    ```
    
    ![Master Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.43.23.png)
    
    Master Server
    
    Alacağımız çıktının en çok olduğu ve üzerine rahatça yorumlar yapabileceğimiz script budur.
    
    ![Secondary Server](MSSQL%20with%20Pacemaker%206824b60f21864402b5c14c1a7dde6af8/Ekran_Resmi_2022-12-18_16.43.38.png)
    
    Secondary Server
    
    Aynı script ikincil bir sunucuda çalıştırıldığında sadece kendisi ile ilgili bilgileri döndürecektir.
    
- **Pacemaker**
    
    Birden fazla linux sunucuyu işletim sistemi seviyesinde kontrolü ve kapsamlı bir şekilde cluster yapma imkanı sağlayan bir araçtır.
    
    Pacemaker kurulumu için indirilmesi gereken paketler..
    
    ```bash
    sudo apt-get install pacemaker pcs fence-agents resource-agents -y
    ```
    
    Kurulumlar tamamlandığında `hacluster` adında bir kullanıcıya sahip oluyoruz. Bu kullanıcı pacemaker’ı servis seviyesinde yönetmemizi sağlıyor.
    
    ```bash
    sudo passwd hacluster
    ```
    
    Parolayı sıfırladıktan sonra servisleri yeniden başlatalım.
    
    ```bash
    sudo systemctl enable pcsd
    sudo systemctl start pcsd
    sudo systemctl enable pacemaker
    ```
    
- **Pacemaker Cluster**
    
    Cluster oluşturmadan önce var olan tüm ayarları(kurulum sırasında otomatik oluşturur) sıfırlayıp pacemaker’ı yeniden başlatalım.
    
    ```bash
    sudo pcs cluster destroy
    sudo systemctl restart pacemaker
    ```
    
    Sonrasında elimizin hostlara erişebilmesi için pacemaker’a yetki verelim.
    
    ```bash
    sudo pcs host auth pace221 pace222 pace223 -u hacluster -p Tele1234!
    pace221: Authorized
    pace223: Authorized
    pace222: Authorized
    ```
    
    Tüm çıktılarda `Authorized` ibaresini gördüysek eğer cluster oluşturabiliriz. Burada önemli husus oluşturacağımız cluster ismi ile MS SQL’de oluşturulan Availability Group isminin aynı olması gerekmektedir. 
    
    ```bash
    sudo pcs cluster setup ag0 pace221 pace222 pace223
    No addresses specified for host 'pace221', using 'pace221'
    No addresses specified for host 'pace222', using 'pace222'
    No addresses specified for host 'pace223', using 'pace223'
    Destroying cluster on hosts: 'pace221', 'pace222', 'pace223'...
    pace223: Successfully destroyed cluster
    pace221: Successfully destroyed cluster
    pace222: Successfully destroyed cluster
    Requesting remove 'pcsd settings' from 'pace221', 'pace222', 'pace223'
    pace221: successful removal of the file 'pcsd settings'
    pace223: successful removal of the file 'pcsd settings'
    pace222: successful removal of the file 'pcsd settings'
    Sending 'corosync authkey', 'pacemaker authkey' to 'pace221', 'pace222', 'pace223'
    pace223: successful distribution of the file 'corosync authkey'
    pace223: successful distribution of the file 'pacemaker authkey'
    pace221: successful distribution of the file 'corosync authkey'
    pace221: successful distribution of the file 'pacemaker authkey'
    pace222: successful distribution of the file 'corosync authkey'
    pace222: successful distribution of the file 'pacemaker authkey'
    Sending 'corosync.conf' to 'pace221', 'pace222', 'pace223'
    pace221: successful distribution of the file 'corosync.conf'
    pace223: successful distribution of the file 'corosync.conf'
    pace222: successful distribution of the file 'corosync.conf'
    Cluster has been successfully set up.
    ```
    
    Son olarak servisi tüm sunucularda başlangıçta çalışacak hale getiriyoruz.
    
    ```bash
    sudo pcs cluster start --all
    sudo pcs cluster enable --all
    ```
    
    Bu işlemden sonra pcs durumuna baktığımızda aşağıdaki çıktıyı göreceğiz.
    
    ```bash
    sudo pcs status
    Cluster name: ag0
    
    WARNINGS:
    No stonith devices and stonith-enabled is not false
    
    Cluster Summary:
      * Stack: corosync
      * Current DC: pace222 (version 2.0.3-4b1f869f0f) - partition with quorum
      * Last updated: Sun Dec 18 17:21:21 2022
      * Last change:  Sun Dec 18 17:20:31 2022 by hacluster via crmd on pace222
      * 3 nodes configured
      * 0 resource instances configured
    
    Node List:
      * Online: [ pace221 pace222 pace223 ]
    
    Full List of Resources:
      * No resources
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
- **Pacemaker** **Stonith**
    
    Pacemaker içerisindeki kaynakları daha detaylı takip edebilmek için stonith’in sağladığı hazır ve hazırlanabilen konfigürasyonlar aracılığı ile kaynak kontrolü ve yönetimi daha detaylı olarak yapılabilir. 
    
    Yukarıdaki son komut çıktısındaki `WARNINGS` kısmında da görülebileceği üzere açık olduğunda aktif olarak bir yapı bulunması gereken bir bileşendir ve kullanmayacaksak kapatmamız gerekmektedir.
    
    ```bash
    sudo pcs property set stonith-enabled=false
    ```
    
    Kapattıktan sonra ise çıktıdan uyarı ifadesi kaybolacaktır.
    
    ```bash
    sudo pcs status
    Cluster name: ag0
    Cluster Summary:
      * Stack: corosync
      * Current DC: pace222 (version 2.0.3-4b1f869f0f) - partition with quorum
      * Last updated: Sun Dec 18 17:24:00 2022
      * Last change:  Sun Dec 18 17:23:58 2022 by root via cibadmin on pace221
      * 3 nodes configured
      * 0 resource instances configured
    
    Node List:
      * Online: [ pace221 pace222 pace223 ]
    
    Full List of Resources:
      * No resources
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
- **Pacemaker Recheck**
    
    Kaynakların ne aralıklarla kontrol edileceği ile ilgili ayarların yapılandırıldığı kısımdır.
    
    ```bash
    sudo pcs property set cluster-recheck-interval=2min
    sudo pcs property set start-failure-is-fatal=true
    ```
    
    ```bash
    sudo pcs resource update ag0 meta failure-timeout=60s
    ```
    
- **Pacemaker MS SQL Resource**
    
    Burada daha önceden MSSQL üzerinde oluşturduğumuz availability group ile pacemaker ile oluşturduğumuz cluster’ı birbirine eşliyoruz.
    
    ```bash
    sudo pcs resource create ag_cluster ocf:mssql:ag ag_name=ag0 meta failure-timeout=30s promotable notify=true
    Error: Agent 'ocf:mssql:ag' is not installed or does not provide valid metadata: Metadata query for ocf:mssql:ag failed: Input/output error, use --force to override
    ```
    
    Burada verdiği hata `mssql-server-ha` paketinin yüklü olmamasından kaynaklıdır. Bu paketi yükledikten sonra aynı komut sorun çıkarmayacaktır.
    
    ```bash
    sudo apt-get install mssql-server-ha
    sudo systemctl restart pacemaker
    ```
    
    Tüm sunuculara paketi kurup pacemaker’ı yeniden başlattıktan sonra tekrar resource oluşturmayı deniyoruz.
    
    ```bash
    sudo pcs resource create ag_cluster ocf:mssql:ag ag_name=ag0 meta failure-timeout=30s promotable notify=true
    sudo pcs status
    Cluster name: ag0
    Cluster Summary:
      * Stack: corosync
      * Current DC: pace222 (version 2.0.3-4b1f869f0f) - partition with quorum
      * Last updated: Sun Dec 18 17:40:04 2022
      * Last change:  Sun Dec 18 17:39:58 2022 by root via cibadmin on pace221
      * 3 nodes configured
      * 3 resource instances configured
    
    Node List:
      * Online: [ pace221 pace222 pace223 ]
    
    Full List of Resources:
      * Clone Set: ag_cluster-clone [ag_cluster] (promotable):
        * ag_cluster        (ocf::mssql:ag):         Slave pace222 (Monitoring)
        * ag_cluster        (ocf::mssql:ag):         Slave pace223 (Monitoring)
        * ag_cluster        (ocf::mssql:ag):         Master pace221 (Monitoring)
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
    İlk aşamada tüm sunucular izleme moduna alınıyor. Üzerlerindeki servisler ve paketler kontrol ediliyor. Sonrasında ise aralarından bir master iki slave seçildiğini göreceksiniz. Sonuç olarak aşağıdaki hale gelecektir.
    
    ```bash
    sudo pcs status
    Cluster name: ag0
    Cluster Summary:
      * Stack: corosync
      * Current DC: pace222 (version 2.0.3-4b1f869f0f) - partition with quorum
      * Last updated: Sun Dec 18 17:41:23 2022
      * Last change:  Sun Dec 18 17:39:58 2022 by root via cibadmin on pace221
      * 3 nodes configured
      * 3 resource instances configured
    
    Node List:
      * Online: [ pace221 pace222 pace223 ]
    
    Full List of Resources:
      * Clone Set: ag_cluster-clone [ag_cluster] (promotable):
        * Masters: [ pace221 ]
        * Slaves: [ pace222 pace223 ]
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
    ```bash
    sudo pcs resource config
     Clone: ag_cluster-clone
      Meta Attrs: notify=true promotable=true
      Resource: ag_cluster (class=ocf provider=mssql type=ag)
       Attributes: ag_name=ag0
       Meta Attrs: failure-timeout=30s
       Operations: demote interval=0s timeout=10 (ag_cluster-demote-interval-0s)
                   monitor interval=10 timeout=60 (ag_cluster-monitor-interval-10)
                   monitor interval=11 role=Master timeout=60 (ag_cluster-monitor-interval-11)
                   monitor interval=12 role=Slave timeout=60 (ag_cluster-monitor-interval-12)
                   notify interval=0s timeout=60 (ag_cluster-notify-interval-0s)
                   promote interval=0s timeout=60 (ag_cluster-promote-interval-0s)
                   start interval=0s timeout=60 (ag_cluster-start-interval-0s)
                   stop interval=0s timeout=10 (ag_cluster-stop-interval-0s)
    ```
    
- **Pacemaker Virtual IP Resource**
    
    Sunuculardan erişim için ihtiyaç olacak olan ortak IP atamasını yapmak üzere sanal IP kaynağı oluşturuyoruz.
    
    ```bash
    sudo pcs resource create mssql_vip ocf:heartbeat:IPaddr2 ip=10.34.74.220
    ```
    
    Bunu yaptığımızda pacemaker’ın VIP’yi mevcut master sunucu üzerine eklediğini görmekteyiz.
    
    ```bash
    sudo pcs status
    Cluster name: ag0
    Cluster Summary:
      * Stack: corosync
      * Current DC: pace222 (version 2.0.3-4b1f869f0f) - partition with quorum
      * Last updated: Sun Dec 18 17:48:17 2022
      * Last change:  Sun Dec 18 17:47:42 2022 by root via cibadmin on pace221
      * 3 nodes configured
      * 4 resource instances configured
    
    Node List:
      * Online: [ pace221 pace222 pace223 ]
    
    Full List of Resources:
      * Clone Set: ag_cluster-clone [ag_cluster] (promotable):
        * Masters: [ pace221 ]
        * Slaves: [ pace222 pace223 ]
      * mssql_vip   (ocf::heartbeat:IPaddr2):        Started pace221
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
    ```
    
    ```bash
    sudo pcs resource config
     Clone: ag_cluster-clone
      Meta Attrs: notify=true promotable=true
      Resource: ag_cluster (class=ocf provider=mssql type=ag)
       Attributes: ag_name=ag0
       Meta Attrs: failure-timeout=30s
       Operations: demote interval=0s timeout=10 (ag_cluster-demote-interval-0s)
                   monitor interval=10 timeout=60 (ag_cluster-monitor-interval-10)
                   monitor interval=11 role=Master timeout=60 (ag_cluster-monitor-interval-11)
                   monitor interval=12 role=Slave timeout=60 (ag_cluster-monitor-interval-12)
                   notify interval=0s timeout=60 (ag_cluster-notify-interval-0s)
                   promote interval=0s timeout=60 (ag_cluster-promote-interval-0s)
                   start interval=0s timeout=60 (ag_cluster-start-interval-0s)
                   stop interval=0s timeout=10 (ag_cluster-stop-interval-0s)
     Resource: mssql_vip (class=ocf provider=heartbeat type=IPaddr2)
      Attributes: ip=10.34.74.220
      Operations: monitor interval=10s timeout=20s (mssql_vip-monitor-interval-10s)
                  start interval=0s timeout=20s (mssql_vip-start-interval-0s)
                  stop interval=0s timeout=20s (mssql_vip-stop-interval-0s)
    ```
    
    ```bash
    ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       ..
    2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:50:56:9c:e0:00 brd ff:ff:ff:ff:ff:ff
        inet 10.34.74.221/24 brd 10.34.74.255 scope global ens160
           valid_lft forever preferred_lft forever
        inet 10.34.74.220/24 brd 10.34.74.255 scope global secondary ens160
           valid_lft forever preferred_lft forever
        inet6 fe80::250:56ff:fe9c:e000/64 scope link 
           valid_lft forever preferred_lft forever
    ```
    
- **Pacemaker Colocation**
    
    Oluşturduğumuz VIP kaynağını master olan node’da konumlandırılması üzerine atama ayarlarını yapıyoruz.
    
    ```bash
    sudo pcs constraint colocation add mssql_vip with master ag_cluster-clone INFINITY
    ```
    
    ```bash
    sudo pcs constraint list
    Location Constraints:
    Ordering Constraints:
    Colocation Constraints:
      mssql_vip with ag_cluster-clone (score:INFINITY) (rsc-role:Started) (with-rsc-role:Master)
    Ticket Constraints:
    ```
    
- **Pacemaker Ordering**
    
    VIP hangi sırada değişeceği ile ilgili tanımı buradan yapıyoruz.
    
    ```bash
    sudo pcs constraint order promote ag_cluster-clone then start mssql_vip
    Adding ag_cluster-clone mssql_vip (kind: Mandatory) (Options: first-action=promote then-action=start)
    ```
    
    ```bash
    sudo pcs constraint list
    Location Constraints:
    Ordering Constraints:
      promote ag_cluster-clone then start mssql_vip (kind:Mandatory)
    Colocation Constraints:
      mssql_vip with ag_cluster-clone (score:INFINITY) (rsc-role:Started) (with-rsc-role:Master)
    Ticket Constraints:
    ```
    
- **Pacemaker Manual Failover**
    
    Elle master değiştirmek için aşağıdaki komutu değiştirebiliriz.
    
    ```bash
    sudo pcs resource move ag_cluster-clone pace223 --master
    ```
    
    Komutu yürüttüğümüzde önce elimizdeki VIP adresimizin paylaşılması durduruluyor sonrasında master sunucumuz slave kısmına taşınıyor. Sonrasında promote edilecek olan sunucu slave arasından çıkartılıyor. Master ataması yapıldıktan sonra daha önceki master’da bir sorun olup olmadığına dair sistem monitor ediliyor. Aynı zamanda master sunucu üzerinde sanal ip adresimiz başlatılıyor. Bu süreç yaklaşık 20 saniye sürmektedir. 
    
    Ancak elle tetiklenmesi yerine master olan sunucunun yeniden başlaması durumunda bir VIP belirlenmesi süreci öncelikle master düştüğü için değişmekte sonrasında master tekrar kullanılabilir hale geldiğinde yerine atadığı makineden koparak yeniden başlayan makineyi VIP olarak promote etmektedir. Bu durumda süreç 90 saniye sürmektedir. 
    
    Makine yeniden başlayıp master ilk olduğu konuma taşındığında ve bu konumda aradan geçen zamandan ötürü oluşabilecek veri eksikliği yahut bir servis sorunundan ötürü çalışmaması ve sonrasında farklı bir sunucu üzerine aktarılması gibi bir durum da söz konusu olabilir. Bu durumda süreç 120 saniye sürmektedir. 
    
    ```bash
    sudo pcs constraint list
    Location Constraints:
      Resource: ag_cluster-clone
        Enabled on:
          Node: pace223 (score:INFINITY) (role:Master)
    Ordering Constraints:
      promote ag_cluster-clone then start mssql_vip (kind:Mandatory)
    Colocation Constraints:
      mssql_vip with ag_cluster-clone (score:INFINITY) (rsc-role:Started) (with-rsc-role:Master)
    Ticket Constraints:
    ```
    

- **Kaynaklar**
    
    [SQL Kurulumu](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-ubuntu?view=sql-server-ver16)
    
    [MSSQL Availability Group Konfigürasyonu](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-configure-ha?view=sql-server-ver16)
    
    [Availability Group Komutları](https://sqlundercover.com/2017/09/19/7-ways-to-query-always-on-availability-groups-using-sql/)
    
    [Failover](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-failover-ha?view=sql-server-ver16)
    
    [Operate](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-availability-group-operate-ha?view=sql-server-ver16)
    
    [Stonith](https://github.com/microsoft/sql-server-samples/tree/master/samples/features/high%20availability/Linux/Cluster%20Configuration/STONITH)
    
    [https://www.adminscout.net/en/os/Ubuntu/osv/20.04/conf/configuration-of-a-cluster-corosync-and-pacemaker-on-an-ubuntu-20-04-server/e1bfcf0dec05488e6be06e1701d489e5.html](https://www.adminscout.net/en/os/Ubuntu/osv/20.04/conf/configuration-of-a-cluster-corosync-and-pacemaker-on-an-ubuntu-20-04-server/e1bfcf0dec05488e6be06e1701d489e5.html)
    
    [https://houseofbrick.com/blog/sql-server-on-linux-install-the-basics/](https://houseofbrick.com/blog/sql-server-on-linux-install-the-basics/)
    
    [https://www.sqlshack.com/manage-sql-databases-in-centos-install-sql-server-on-centos/](https://www.sqlshack.com/manage-sql-databases-in-centos-install-sql-server-on-centos/)
    
    [https://github.com/MicrosoftDocs/sql-docs/blob/live/docs/linux/sql-server-linux-setup-machine-learning.md](https://github.com/MicrosoftDocs/sql-docs/blob/live/docs/linux/sql-server-linux-setup-machine-learning.md)