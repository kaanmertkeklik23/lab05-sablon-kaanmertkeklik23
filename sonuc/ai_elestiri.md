# Yapay Zeka Eleştirisi — Föy 05

**Öğrenci No:** 250541053  

### 1. Yapay Zekaya Sorduğum Soru
"PostgreSQL’de SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED dersem kirli okuma yapabilir miyim? Kısaca açıkla."

### 2. Modelin Verdiği Cevap
"Evet, READ UNCOMMITTED seviyesinde kirli okuma (dirty read) gerçekleşir. Bu seviyede çalışan bir işlem, diğer kullanıcıların henüz COMMIT etmediği verileri okuyabilir. Dolayısıyla başka bir oturumda yapılan ama geri alınabilecek değişiklikleri veritabanında görebilirsiniz."

### 3. Terminalde Yaptığım Deneyin Transkripti

```sql
-- Oturum B: READ UNCOMMITTED seviyesinde başlattım
vtysdb=> BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN
vtysdb=*> SHOW transaction_isolation;
 transaction_isolation 
-----------------------
 read uncommitted
(1 row)

-- Oturum A: Üye 3'ün kredisini 77 artırdı ama henüz COMMIT etmedi
vtysdb=> BEGIN;
BEGIN
vtysdb=*> UPDATE krediler SET kredi = kredi + 77 WHERE uye_id = 3;
UPDATE 1
vtysdb=*> SELECT kredi FROM krediler WHERE uye_id = 3;
 kredi 
-------
   867
(1 row)

-- Oturum B: Aynı anda üye 3'ü sorguladı
vtysdb=*> SELECT kredi FROM krediler WHERE uye_id = 3;
 kredi 
-------
   790
(1 row)

-- Oturum A: İşlemi iptal etti
vtysdb=*> ROLLBACK;
ROLLBACK

-- Oturum B: Bitirdi
vtysdb=*> COMMIT;
COMMIT
```

### 4. Deney Gözlemim ve Yapay Zekaya Eleştirilerim

Modelin cevabını terminalde psql oturumlarında bizzat test ettiğimde yapay zekanın tamamen yanıldığını gördüm. Yazdığı açıklamadaki sorunlar şunlar:

İlk olarak, model sanki bütün veritabanları aynı mantıkla çalışıyormuş gibi genel SQL teorisini ezbere tekrarlamış ve PostgreSQL'e özel davranışı tamamen atlamış. PostgreSQL'de `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` komutunu çalıştırdığımızda sistem hata vermeden kabul ediyor hatta `SHOW transaction_isolation` dediğimizde ekranda `read uncommitted` yazıyor. Fakat arka planda PostgreSQL bu seviyeyi otomatik olarak `READ COMMITTED` seviyesine çekiyor. MVCC mimarisinden dolayı PostgreSQL'de dirty read fiziksel olarak mümkün bile değil, yani yapay zekanın "commit edilmemiş veriyi görebilirsiniz" iddiası tamamen yanlış çıktı.

İkinci olarak, transkriptimde görüldüğü üzere Oturum A üye 3'ün kredisini 790'dan 867'ye çıkardığında, Oturum B `READ UNCOMMITTED` modunda olmasına rağmen 867 değerini göremedi, yine eski ve onaylı değer olan 790'ı gördü. Yapay zeka motorun bu güvenlik mekanizmasını ve sürümler arası izolasyonu bilmeden yüzeysel konuştuğu için geliştiriciyi ciddi şekilde yanıltıyor. PostgreSQL belgelerini ve pratik çıktıyı kontrol etmeden genel kitap bilgisini doğrudan gerçekmiş gibi sunması en büyük eksikliği.
