---
layout: post
title: OSI ve TCP/IP Modelleri
date: 2023-06-14 23:08 +0300
categories: [Network Notlarım, Network Modelleri]
tags: [networking, osi-katmanları, tcp-ip-katmanları]
---

OSI ve TCP/IP modelleri, bilgisayar ağlarının karmaşıklığını anlamak ve yönetmek için geliştirilmiş iki temel iletişim modelidir. 

OSI modeli, Uluslararası Standartlar Organizasyonu (ISO) tarafından oluşturulan bir standarttır ve bilgisayar ağlarının işleyişini yedi katmana böler. Bu katmanlar, veri iletişiminin her bir aşamasını temsil eder ve iletişimin sorunsuz bir şekilde gerçekleşmesini sağlamak için ağ bileşenlerinin görevlerini tanımlar.

Öte yandan, TCP/IP modeli, internetin temelini oluşturan protokollerin bir kümesini tanımlar ve dört katmandan oluşur. İnternetin ana iletişim protokolüdür ve dünya çapında bilgisayar ağlarının birbirleriyle iletişim kurabilmesini sağlar.

Bu modeller, bilgisayar ağlarının karmaşıklığını azaltmak, standartları belirlemek ve iletişimde sorunları tanımlamak için önemli bir rol oynar. Her iki model de ağ mühendislerine ve uzmanlara, ağ bileşenlerini ve protokolleri anlamaları, sorun giderme yapabilmeleri ve veri iletişiminin sorunsuz bir şekilde gerçekleşmesini sağlayabilmeleri için bir rehber sağlar.

# OSI Modeli

OSI modelinin katmanları yazılımsal/donanımsal ve upper/lower olarak ayrıca bir de kalbi olarak belirtilen katmanlara ayrılmıştır. Bunu anlatan diyagramı aşağıda görebilirsiniz

![osi_layers_01.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/osi_layers_01.png)

Bu modelin katmanlarını yazının devamında okuduğunuzda neden hardware/software veya upper/lower vs. bölümlere ayrıldığını kendiniz anlayabileceksiniz. 

## 1. Physical Layer (Fiziksel Katman)

Fiziksel katman, bitlerin kablolar ve sinyaller aracılığıyla iletimini sağlar.

- Verinin iletilmesi için sinyal türü burada belirlenir

- Cihazların fiziksel olarak nasıl bağlanılacağı burada belirlenir

- İki cihaz arası transmission mode'u belirlenir

Kısacası fiziksel olan şeyler burada belirlenir: Kablo türleri, topoloji türleri sinyaller vs.

## 2. Data Link Layer (Veri Bağlantı Katmanı)

Data Link katmanı; switch, bridge gibi cihazlar yardımıyla veri iletişimini sağlamaktan sorumludur. Bu katman iki alt katmana ayrılır:

- **Logical Link Control Layer (LLC Layer)**
  
  LLC Layer, Data Link Layer'ın üst katmanıdır ve data frame'lerin alıcı tarafında ağ katmanına aktarılmasından sorumludur. Bu katman, data frame'lerin doğru ve hatasız iletimini sağlar. LLC Layer, data frame'lerideki ağ katmanı protokolünün adresini tanımlar ve akış kontrolü gibi işlevleri gerçekleştirir.

- **Media Access Control Layer (MAC Layer)**
  
  Data Link Layer'ın alt katmanıdır ve mantıksal bağlantıyı fiziksel ortamla ilişkilendirir. Bu katman, data frame'lerin fiziksel ortam üzerinde iletilmesini sağlar ve iletilecek frame ile alakalı iletişimde kullanılan kontrol bilgilerini yönetir. Ayrıca, birden çok cihazın aynı fiziksel ortamı paylaştığı durumlarda erişim kontrolünü sağlar.

Kısacası LLC Layer, ağ katmanının üzerindeki iletişimi yönetirken, MAC Layer, fiziksel ortamın üzerindeki iletişimi yönetir. Bu alt katmanlar, data frame'leri hatasız ve verimli bir şekilde iletilmesinden sorumludur.

## 3. Network Layer (Ağ Katmanı)

Network Layer, cihazların ağ üzerinde benzersiz adreslerle tanımlanmasını yönetir ve her cihaza bir IP adresi atanır. Bu adresler sayesinde cihazlar ağ üzerinde izlenebilir ve bulunabilir hale gelir.

Bu katmanda verileri küçük paketlere ayırır ve hedef cihaza iletmek için uygun protokolleri kullanır. Her paket, kaynak ve hedef IP adresleriyle birlikte taşınır ve ağ üzerinde doğru yönlendirme yapılır. Bu katmanda kullanılan yaygın protokoller arasında IP (Internet Protocol) ve IPv6 (Internet Protocol version 6) kullanılır. Bununla birlikte ICMP ve IGMP protokolleri de kullanılabilir.

## 4. Transport Layer (Taşıma Katmanı)

![transport_layer_wallpaper_01.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/transport_layer_wallpaper_01.png)

Transport Layer, uygulama katmanı ile ağ katmanı arasında veri iletimini yöneterek iletişim sağlar. Bu katmanda kullanılan yaygın protokoller arasında TCP (Transmission Control Protocol) ve UDP (User Datagram Protocol) bulunur. TCP, güvenilir ve sıralı veri iletimi sağlarken, UDP daha hızlı ancak güvensiz bir iletişim sunar.

Bunun yanında SSL, TLS, SCTP protokolleri de kullanılabilir.

## 5. Session Layer (Oturum Katmanı)

Bu katman, iki cihaz arasında iletişimin açılması ve kapanmasından sorumludur. İletişimin açılıp kapanması arasındaki süreye oturum denir. Oturum katmanı, iletişimdeki tüm verilerin aktarılması için oturumun yeterince açık kalmasını sağlar ve ardından kaynakların boşa harcanmaması için oturumu hızlı bir şekilde kapatır.

Oturum Katmanı, uygulama katmanıyla transport katmanı arasında uygulamalar için iletişim sağlar. Bu katmanda kullanılan protokoller arasında NetBIOS, RPC  ve SSH bulunabilir.

## 6. Presentation Layer (Sunum Katmanı)

Sunum katmanı, verileri formatlama, encode etme, şifreleme ve çözme gibi gibi kullanıcı için gösterilecek verilerden sorumludur.

Uygulama katmanıyla ve altında bulunan Session Layer ve Transport Layer ile iletişim sağlar. Genel olarak application layer'den gelen verileri şifreleyerek gönderirken, alt katmandan gelen verileri decode edilerek gönderir.

Kısacası verilerin anlaşılabilir data'lara çevrilmesinden sorumludur. Bu veriler ses, video, resim, yazı vs. olabilir.

## 7. Application Layer (Uygulama Katmanı)

Kullanıcıdan gelen veya alt katmanlardan gelen verileri farklı uygulamalar arasında veri iletişimini sağlar. Bu katman kullanıcıdan veri alınabilen tek katmandır.

Bu katmanda FTP, HTTP/S, SMPT, TELNET gibi protokoller kullanılabilir.

## OSI Modeli Nasıl İşler?

OSI modelinde veri akışı, katmanlar arasında gerçekleşen iletişimi tanımlar. Bu modelde veri, bir cihazdan diğerine doğru bir yönde ilerler.

![osi_works_01.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/osi_works_01.png)

Aşağıda bir kullanıcı tarafından girilen veri başka bir kullanıcıya iletimi adım adım anlatılmıştır:

1. Veri, kaynak cihazdan başlayarak üst katman olan Application Layer (Uygulama Katmanı) ile iletişime girer.
2. Uygulama Katmanı, veriyi alt katmanlara (Presentation Layer, Session Layer) aktarır.
3. Presentation Layer, veriyi alt katmanlara (Session Layer, Transport Layer) iletmek için gerekli dönüştürme, şifreleme ve sıkıştırma işlemlerini gerçekleştirir.
4. Session Layer, veriyi alt katmanlara (Transport Layer, Network Layer) aktarır ve iletişim oturumunu başlatır veya sonlandırır.
5. Transport Layer, veriyi alt katmanlara (Network Layer, Data Link Layer) iletir ve verinin hedefe güvenli ve hatasız bir şekilde iletilmesini sağlar.
6. Network Layer, veriyi alt katmanlara (Data Link Layer, Physical Layer) aktarır ve ağ üzerinden yönlendirme işlemlerini gerçekleştirir.
7. Data Link Layer, veriyi alt katmanlara (Physical Layer) aktarır ve fiziksel bağlantıyı sağlar.
8. Physical Layer, veriyi kablolardan veya diğer fiziksel ortamlardan ileterek hedef cihaza ulaştırır.

## Encapsulation ve De-encapsulation

Encapsulation ve de-encapsulation mantığı, verinin OSI modelinin katmanları arasında taşınması ve işlenmesi için bir yol sağlar. Her katman, kendi başlığını ekleyerek veya çıkararak, veriyi kendi ihtiyaçlarına göre şekillendirir ve alt katmana aktarır. Bu sayede, veri katmanlarda işlenirken her katmanın belirli bir görevi yerine getirmesi sağlanır ve sonunda hedef uygulamaya ulaşır. Bu süreç, veri iletişiminin düzenli ve yapılandırılmış bir şekilde gerçekleştirilmesini sağlar.

![osi_enc_de_enc_01.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/osi_enc_de_enc_01.png)

1. **Encapsulation (Kapsülleme):**
   Verinin her katmanda özel bir veri birimi veya "kapsül" içinde taşınmasını ifade eder. Gönderici cihazdaki uygulama katmanından başlayarak, veri katmanlara doğru ilerlerken her katmanda kendi veri birimine eklenir. Her katman, kendine ait bir başlık (header) veya ek bilgiler (metadata) ekler ve bu şekilde veri birimleri katmanlar arasında taşınır. Bu süreç katmandan katmana aşağı doğru ilerlerken her katman, kendine ait işlevleri gerçekleştirir ve daha önceki katmanın verisini alır, başlığını ekler ve alt katmana gönderir. Böylece, veri ilgili katmanın kapsülü içinde taşınmış olur.

2. **De-encapsulation (Kapsülün Çözülmesi):**
   Alıcı cihazda gerçekleşen bir süreçtir ve verinin katmanlar arasında çözülerek hedef uygulamaya ulaşmasını sağlar. Alıcı cihazdaki fiziksel katmandan başlayarak yukarı doğru ilerlerken her katmanda kendi veri birimini çözer. Her katman, kendi başlığını veya ek bilgilerini çıkarır ve alt katmana geçirir. Bu şekilde, veri katmandan katmana yukarı doğru çözülerek ilerler. En sonunda, veri Application Layer (Uygulama Katmanı)'na ulaşır ve hedef uygulama tarafından kullanılabilir hale gelir.

Örneğin wireshark aracı ile ağda geçen paketleri incelediğimizde:

![wireshark_traffic_01.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/wireshark_traffic_01.png)

Herhangi bir paket'e structure view ile baktığımızda:

![encapsuled_data_s.png](/assets/img/2023-06-14-osi-ve-tcp-ip-modelleri/encapsuled_data_s_01.png)

Burada her bir layer'den gelen verilerin dizilimini görebilirsiniz. Sonraki yazılarımda TCP, UDP, DNS vs. protokollerin yapısını ve çalışma mantığını anlatabilirim. 

# TCP/IP Modeli

## 1. Network Layer (Ağ Katmanı)

## 2. Internet Layer (İnternet Katmanı)

## 3. Transport Layer (Taşıma Katmanı)

## 4. Application Layer (Uygulama Katmanı)

# OSI ve TCP/IP Karşılaştırılması
