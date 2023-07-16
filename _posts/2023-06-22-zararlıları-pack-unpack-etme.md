---
layout: post
title: Zararlıları Pack/Unpack Etme
date: 2023-06-22 20:32 +0300
categories: [Malware Analizi, Tutorial Serisi]
tags: [packing, unpacking]
---

Pack'leme işlemi genel olarak statik analizlerde (dinamik analizler için de olabilir)
program kodlarını karmaşıklaştırarak tersine mühendisliği zorlaştırmak ve av'ler tarafından farkedilmemek için yapılır.

> Basit Statik Analizler, packlenmiş dosyalar üzerinde pek kullanışlı olmaz. Önce unpack edilmesi gerekir, daha sonra analizlere devam edilmelidir.

Packer'lar genel olarak şunları yapabilirler:

- Compressing (Sıkıştırma)

- Crypting (Şifreleme)

- Anti Reverse Engineering

- Anti-Disassembly

- Anti-Debugging

- Anti-VM

# Packer Kategorileri

Packleme işlemini yapan programlara packer denir. Packer'ları kendi aralarında kategorize edecek olursak;

- **Compresser:** Orijinal programın boyutunu küçültür.

- **Protector:** Tersine mühendislik yapmayı zorlaştırır.

- **Crypter:** Orijinal programı şifreler.

- **Encoder:** Orijinal programı belirli algoritmalarla şifreler.

- **Bundler:** Bir çok dosyalı programı tek dosya olarak çalıştırır.

- **Mutater:** Program kodunu değiştirir (aynı komut seti ve mimari, ancak değiştirilmiş kod)

- **Virtualiser:** Turns original code into virtual code with embedded virtual machine 

![packer_categories_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/packer_categories_01.png)

Packer'lar, tüm dosyayı pack'leyebilir veya yalnızca text ve data bölümlerini de pack'leyebilir.

# Pack ve Unpack Mantığı

Peki kod karmaşıklaştı tamam da, bilgisayar packli programı nasıl anlayacak? diyebilirsiniz. Packlenmiş programlar execute edildiğinde run-time sırasında memory'de açılır. Bunu sağlayan yapıya da Unpacking Stub deniyor. 

![pack_flow_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/pack_flow_01.png)

Normal bir program pack'leme işleminden geçtikten sonra, oluşan pack'lenmiş programa Upacking Stub denen bir yapı eklenir (bu yapı pe dosyasının herhangi bir yerinde olabilir). Bu yapı tahmin edeceğiniz üzere pack'lenmiş programı eski haline (orijinal programa) çevirmeyi sağlıyor. Eğer packlenmiş programı debugger'a sokup analize kalkışırsanız, artık Entry Point Unacking Stub bölümünü işaret edecektir.

> Entry Point'in ne olduğunu bilmiyorsanız [Windows Portable Executable (PE) Dosyaları](http://alipasaturhan.github.io/posts/windows-portable-executable-pe-dosyalar%C4%B1/#optional-header) yazıma bakabilirsiniz.

Packlenmiş program execute edildiğinde Unpacking Stub, orijinal binaryi çözer (run-time sırasında) ve ardından kontrolü OEP'e (yani Original Entry Point) aktararak orijinal programın yürütülmesini sağlar. Packer'ların bunu yapmasına da "Tail Jump" deniyor.

Unpacking stub aşağıdaki üç adımı sırasıyla gerçekleştirir:

- Orijinal programı belleğe açar.
- Orijinal programın tüm import'larını çözer.
- Yürütme kontrolünü orijinal giriş noktasına (OEP) aktarır.

Packer'lar bu yöntemi çaktırmadan yapmak için de genelde NtContinue veya ZwContniue fonksiyonlarını ret veya call ederek sanki normal bir işlemmiş gibi göstermeye çalışırlar.

> Unpacking stub genellikle küçüktür, çünkü programın ana işlevselliğine katkıda bulunmaz ve işlevi genellikle orijinal programı açmaktır.

# Single, Multi ve Re-Pack Nedir?

Şu anda, tek katmanlı paketleme, yeniden paketleme veya çok katmanlı paketleme algoritmaları, kötü amaçlı yazılımın tespit edilmeden kalmasına yardımcı olmak için kötü amaçlı yazılım geliştirmede yardımcı olmak için kötü amaçlı yazılım geliştirmede yaygın olarak kullanılmaktadır.

![single_multi_re-pack_diagram.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/single_multi_re-pack_diagram_01.png)

## Single Layer Packing

Tek katmanlı paketleme algoritması: Bu paketleyiciler en basit durumu temsil eder. Tek katmanlı paketleme, belirli bir ikili dosyayı paketlemek için yalnızca bir paketleyici kullanır. Bu paketleme tekniği program dosyanın boyutunu, bölüm sayısını ve adını değiştirir (Şekil 2).

## Multi Layer Packing

Çok katmanlı paketleme algoritması: Bu paketleyiciler, her biri bir sonraki rutinin paketini açmak için sırayla çalıştırılan birden fazla paket açma katmanı içerir. Orijinal kod yeniden oluşturulduktan sonra, son geçiş kontrolü ona geri aktarır. Bu paketleme, belirli bir ikiliyi paketlemek için potansiyel olarak farklı paketleyicilerin bir kombinasyonunu kullanır ve aynı giriş ikilisinden çok sayıda paketlenmiş ikilinin oluşturulmasını kapsamlı bir şekilde kolaylaştırır. Çok katmanlı paketleme algoritmaları, tek katmanlı paketlenmiş bir program dosyanın boyutunu, bölüm sayısını ve adını değiştirir.

## Re-Packing

Yeniden paketleme algoritması: Bu paketleyiciler, her biri bölümlerin her birini açmak için sırayla çalıştırılan yeniden paketlenmiş paket açma katmanları içerir. Yeniden paketleme algoritması, belirli bir ikili dosyayı paketlemek için aynı paketleyiciyi iki kez kullanır ve tek katmanlı paketleyiciye benzer sıkıştırma teknikleri kullanır, ancak boyutu değiştirir.

![single_multi_re-pack_diagram_02.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/single_multi_re-pack_diagram_02.png)

> Bu konu hakkında daha detaylı bilgi almak isteyenler için baya emek verilmiş bir yazı paylaşıyorum: [Packer Detection for Multi-Layer Executables Using Entropy Analysis](https://www.mdpi.com/1099-4300/19/3/125)

# Memory'e Map Etme Süreci

Normal program dosyalar yüklendiğinde, bir yükleyici disk üzerindeki PE başlığını okur ve bu başlığa dayanarak yürütüprogramnın her bir bölümü için bellek ayırır. Yükleyici daha sonra bölümleri bellekte ayrılan alanlara kopyalar.

![normal_pe_file_loading_the_memory_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/normal_pe_file_loading_the_memory_01.png)

Paketlenmiş program dosyalar da PE başlığını biçimlendirir, böylece yükleyici bölümler için alan ayırır. Bu bölümler, orijinal programdan gelebilir veya açma öğesi bölümleri oluşturabilir. Açma öğesi, her bir bölüm için kodu açar ve ayrılan alana kopyalar. Kullanılan kesin açma yöntemi, paketleyicinin hedeflerine bağlı olarak değişir ve genellikle açma öğesi içerisinde bulunur.

![unpack_flow_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/unpack_flow_01.png)

Kısacası packlenmemiş programlar işletim sistemi tarafından direkt yüklenir. Fakat packlenmiş programlarda önce unpacking stub yüklenir daha sonra unpacking stub orijinal programı yükler.

# IAT'nin Gizlenme ve Çözülme Süreci

Orijial programlar kullandığı api'leri import tablolarında gösterirken, packlenmiş programlar bu fonksiyonları gizlemek ister (genelde öyle yaparlar, yapmayanları da var). Bu yüzden de programın import tablosunda oynama yaparlar. Eğer packlenmiş bi programın iat'sine bakarsanız sayılı api'leri import ettiğini göreceksiniz (packer'lar genelde belirli bikaç fonksiyonu import eder, ona da geleceğiz).

Packlenmiş program memory'e girdiğinde IAT'sini tekrar ayarlamak zorundadır (bunu da unpacker stub ile yapması gerekiyor). Çünkü işletim sisteminin pe loader'ı packlenmiş bir programın import tablosunu o haliyle okuyamaz.

> Programın IAT'si memory'de tekrar oluşurken bazen zorlu ve zaman alıcı olabilir, ancak programın çalışması için her türlü bu işlem uygulanmak zorunda.

Packer'ların en yaygın kullandığı yöntem; `LoadLibrary` ve `GetProcAddress` fonksiyonlarını kullanarak IAT'yi yeniden oluşturmaktır. LoadLibrary ile dll'i belleğe yükleme çağrısı yapılır, GetProcAddress ile de her import edilen fonksiyon için adres alır.

Packer'lar aşağıdaki yöntemlerle IAT'de oynama yapabilirler

1. Sadece LoadLibrary ve GetProcAddress fonksiyonlarını import ederek IAT'yi çözümleme (en yaygını bu).

2. Orijinal IAT'yi korur. PE Loader zahmetsizce import edilen dll'leri belleğe yükler (fakat sıfır gizlilik).

3. Orijinal IAT'de import edilen her dll için yalnızca bir fonksiyonda oynama yapılmaz (bir öncekine oranla daha gizlidir, fakat belleğe yüklenen modüllerin ismi yine de gözükecektir).

4. Tüm IAT'yi kaldırır (LoadLibrary ve GetProcAddress fonksiyonları da dahil). Bu yöntem en karmaşık ve performans açısından yavaş olan yöntemdir ve unpacker stub'ın küçük bir alan olması gerekirker, haddinden fazla bir alan kaplayabilme ihtimali yüksektir (Fakat gizlilik için en iyi yöntem budur). Daha sonrasında LoadLibrary ve GetProcAddress fonksiyonlarını bulmak zorundadır ve diğer gizlenen fonksiyonları/modülleri de bu fonksiyonlar aracılığıyla çözmelidir.

Örneğin calc.exe'nin hem UPX ile packlenmiş halinin hem de orijinal halinin iat'sine bakalım:

![original_iat_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/original_iat_01.png)

Şimdi de compress edilmiş calc.exe dosyasına bakalım

![packed_iat_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/packed_iat_01.png)

IAT'nin halini görüyorsunuz... Modül sayısı aynı, ve her birinde sadece bir fonksiyon korunmuş. Demek ki UPX aracının 3. packleme yöntemini kullandığını çıkarabiliriz.

# Nedir Bu Entropy?

Entropy programdaki kod düzensizliğin bir grafiği/hesaplamasıdır. Packer'ların amacı da program kodlarını karmaşıklaştırmak olduğundan, programların packlenip packlenmediğini entropy değerinden anlayabiliriz.

Packlenmiş programda, anlamsız byte'lar vardır (unpacker stub için anlamlı tabii). Bu yüzden entropy değerleri yüksek olur. Normal programların kodları daha düzgün bir şekilde olduğu için daha düşük entropy değerlere sahiptirler.

Örneğin yine normal bir calc.exe'nin entropy değerine bakmak istersek:

![original_entropy_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/original_entropy_01.png)

Şimdi de packlenmiş calc.exe'nin entropy grafiğine bakalım:

![packed_entropy_01.png](/assets/img/2023-06-22-zararlıları-pack-unpack-etme/packed_entropy_01.png)

# Programın Packlendiğine İşaretler

Önce başlıkta da belirttiğim gibi program packlenince IAT'sinden belli oluyor. Yani ya çok az api import edilir ya da hiç (ya da istisna, iat'ye dokunulmaz). Özellikle LoadLibrary ve GetProcAddress fonksiyonlarına dikkat edilmesi gerekiyor.

Ayrıyeten OllyDbg'nin program açtığında otomatik analizicisi, programın packlendiğine yönelik bir uyarı çıkarır. IDA'dan açıldığında sadece küçük bir kod miktarı gösterilir.

Programın pe yapısındaki section'ların isimleri normalden farklı bir isime sahiptir veye anormal boyutlara sahip section'ları vardır.

Yine önceki başlıkta bahsettiğim gibi, Entropy grafiği uçmuştur.

PE dosyasının raw size'ı, virtual size'ından küçüktür.

Programın packlenip packlenmediğini analiz eden programlar var. Programlardan en çok bilinenlerin listesi:

- Detect It Easy

- ExeInfoPE

- PEiD

- RDG Packer Detector

- TrIDNet

# Unpack'leme Yöntemleri

Packlenmiş zararlıyı üç ayrı şekilde unpack edebiliriz:

- Automated Static Unpacking

- Automated Dynamic Unpacking

- Manual Dynamic Unpacking

Automated unpacking, manual dinamik unpack'lemeye göre daha hızlı ve kolaydır ama otomatik teknikler her zaman işe yaramaz. Kullanılan packer tipini bulursak otomatik unpacker var mı yok mu bulmamız gerekiyor. Yoksa son çare packlenmiş dosyayı manuel olarak açmak zorundayız.

> Packlenmiş zararlılarla uğraşırken, biz analizciler genelde zararlının işlevlerine odaklanırız. Fakat şöyle bişey var; genelde zararlı unpack edildiğinde orijinal halinin tamamı getirilmez (ama işlevleri orijinaliyle aynıdır).

## Automated Static Unpacking

Packlenmiş bi dosyayı unpack etmenin en hızlı yöntemidir. Dosyayı açmadan tamamen packer'ın algoritmasının tersine giderek unpackleme işlemi uygulanır. Otomatik statik açma programları tek bir paketleyiciye özgüdür, örneğin ASPacker ile packlenmiş bi dosyayı UPX Unpacker ile unpack edemezsiniz.

Örnek unpacker programlarından [TitanMist](https://www.reversinglabs.com/open-source/titanmist.html), popüler packer'ların unpacker'larını tek bi çatı altında toplamaya çalışmıştır.

```powershell
C:\TitanMist>TitanMist.exe -i packed.exe -o unpacked.exe -t python
Match found!
│ Name: UPX
│ Version: 0.8x - 3.x
│ Author: Markus and Laszlo
│ Wiki url: http://kbase.reversinglabs.com/index.php/UPX
│ Description:
Unpacker for UPX 1.x - 3.x packed files
ReversingLabs Corporation / www.reversinglabs.com
[x] Debugger initialized.
[x] Hardware breakpoint set.
[x] Import at 00407000.
[x] Import at[x] Import at 00407008.[Removed]
[x] Import at 00407118.
[x] OEP found: 0x0040259B.
[x] Process dumped.
[x] IAT begin at 0x00407000, size 00000118.
[X] Imports fixed.
[x] No overlay found.
[x] File has been realigned.
[x] File has been unpacked to unpacked.exe.
[x] Exit Code: 0.
Unpacking succeeded! 00407004.
```

## ~~Automated Dynamic Unpacking~~

## ~~Manuel Dynamic Unpacking~~

# ~~Yanlış Anlaşılan Pack'ler~~

# ~~Packed DLL's~~

# Kaynakça ve Referanslar

1. Pratical Malware Analysis, No Strach Press

2. Learn Malware Analysis, Packt 

3. Mastering Malware Analysis, Packt

4. Antivirus Bypass Techniques, Packt

5. The Antivirus Hacker's Handbook, Wiley

6. [Entropy \| Free Full-Text \| Packer Detection for Multi-Layer Executables Using Entropy Analysis](https://www.mdpi.com/1099-4300/19/3/125)
