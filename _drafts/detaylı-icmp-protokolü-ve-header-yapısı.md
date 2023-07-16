---
layout: post
title: Detaylı ICMP Protokolü ve Header Yapısı
date: 2023-06-27 17:53 +0300
categories: [Network Notlarım, Protokoller]
tags: [icmp, icmp-header]
---

Bilgisayarlar kendi aralarında iletişim sağlamak isterlerken bağlantı kopukluğu, data'nın iletim gibi iletişim ortamındaki sorunlar hakkında geri bildirim sağlar, bu da ICMP sayesinde yapılabilir. Yani IP'nin crash reporter'ı gibi bir protokol diyebiliriz. Zaten IP kesinlikle güvenilir değildir, bu yüzden ICMP ve IP ayrılmaz ikili diye geçer. Fakat ICMP'nin amacını doğru anlamak lazım; IP'nin daha güvenilir bir protokol olması için değil, feedback verebilmek için vardır.

# Message Types

ICMP'nin resmen bir geri bildirim protokolü olduğunu söylemiştik. Örneğin bir bilgisayarın ayakta olup olmadığını görmek için Echo (ICMP bunu, mesaj tipi: 8 olarak tanımlar) mesajı kullanabiliriz. 

Kendi içinde birçok feedback verebilen mesaj tipleri barındırıyor. Önemli mesaj tipleri aşağıdaki gibidir:

- 0  Echo Reply

- 3  Destination Unreachable

- 4  Source Quench

- 5  Redirect

- 8  Echo

- 11  Time Exceeded

- 12  Parameter Problem

- 13  Timestamp

- 14  Timestamp Reply

- 15  Information Request

- 16  Information Reply

> Tam listeyi görmek için linke tıklayabilirsiniz: [Internet Control Message Protocol (ICMP) Parameters](https://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml)

# Header fields

IP header'ı bu yazıda anlatmak istemiyorum, zaten buna ayrı bir post yayınlamayı düşünüyorum. Aşağıdaki diyagramda IP header'ın bulunma sebebi, yazının ilk başlarında söylenilen IP-ICMP kardeşliğindendir (yani bu iki header encapsule edilip ICMP paketi oluşturulur) 

![icmp_header_diagram_01.png](/assets/img/2023-06-27-detaylı-icmp-protokolü-ve-header-yapısı/icmp_header_diagram_01.png)

Aşağıda ICMP header alanlarından kısaca bahsetmeye çalıştım.

**Type:** Mesaj tipi kodudur.

**Code:** Mesaj tipine özel olan flag'lerin belirtildiği alandır.

**Checksum:** Bu alana genelde 0  değeri atanmalıdır. Fakat gerçek amacı ICMP datalarının hash'ini tutmaktır.

> Mesaj tiplerine göre farklı alanlar gelmektedir. Örnek vermek gerekirse tip 8 olan Echo mesajında "Identifier" ve "Sequence Number" varken, tip 11 olan Time Exceeded (yani TTL'in 0 olma durumu) mesajında "unused", "IP Version" bölümleri gelir.
> 
> Mesaj tiplerine göre alanların tamamını görebilmek için tıklayınız: 

TREE OLARAK YAP (HER MESAJ TİPİNE ÖZEL FARKLI ALANLAR ÇIKIYOR)

# Message Type'lara göre ICMP Paketleri

ICMP header diyagramını incelediğimizde Type alanının 8 bit olduğunu görüyoruz. O zaman mantıken 256 tane mesaj tipinin olduğu kanısına varabiliriz (tam listeyi ikinci başlık olan "Message Types" da verdim).



## Echo Reply (0)

## Destination Unreachable (3)

## Source Quench (4)

## Redirect (5)

## Echo (8)

## Time Exceeded (11)

## Parameter Problem (12)

## Timestamp (13)

## Timestamp Reply (14)

## Information Request (15)

## Imformation Reply (16)

bitişinde TraceRoute ve Ping'leme nedir/nasıl çalışır adlı post linkini ekle

# Kaynakça ve Referanslar
