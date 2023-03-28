---
layout: post
title: Derinlemesine Windows Objects ve Handles (0x02)
date: 2023-03-16 22:28 +0300
categories: [Reverse Engineering, Windows]
tags: [reversing, windows, internal, object, handle]
---

![banner](/assets/img/2023-03-16-derinlemesine-windows-objects-ve-handles-0x02/banner.png)

# Object Structure

Objelerin her birinin aynı structure'da olduğunu ve sadece birer bellek parçalarından ibaret olduğundan önceki yazıda bahsetmiştik. Her obje kabaca header ve body kısımlarından oluşur. Object manager sadece nesnenin header'ını ve footer'ını yönetir ve body, nesne türüne göre bir executive sub-system tarafından yönetilir. Örneğin, bir file objesini, executive sub-system olan I/O manager tarafından yönetilir. 

Her objenin header'ı vardır ve aynı yapıdadır, fakat her objenin body kısmı aynı değildir. File (FILE_OBJECT), Device (DEVICE_OBJECT), Process (EPROCESS) vb. aslında body kısmındaki verilerdir ve bir objenin tipi ise objenin header'ında tutulan verilerle anlaşılır. Aşağıda objelerin genel yapısının diyagramı verilmiştir:

<div>
<iframe width="100%" height="550" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7Vttc6o4FP41zux%2B2A4vgvrRF%2Fqy2yu91c72fuqkkgq3SNwYW72%2FfhNIlEC0tIXKeHU6U3LyAjnPeU5CzqFh9merCwzm%2FjfkwbBhaN6qYQ4ahqHTP%2FqPSdaJxLY7iWCKA4832gpGwS%2FIhRqXLgMPLqSGBKGQBHNZOEFRBCdEkgGM0avc7AmF8l3nYApzgtEEhHnpv4FH%2FETatrSt%2FBIGU1%2FcWdd4zSOYPE8xWkb8fg3D1OJfUj0DYiwuWPjAQ68pkek0zD5GiCRXs1Ufhky3Qm1Jv%2FMdtZvnxjAiRTq8kMsOiIaBc3N9b9%2B9gOfBP7O%2F9FYyzAsIl1DMI35ashYaiucI2Sh6w%2By9%2BgGBozmYsNpXahNU5pNZyKufgjDsoxBhWo5QRBv1FgSjZyiEVE3n8Y%2FWADzhBmHTUn5CfI4vEBO4Son4BC8gmkGC17SJqO1YSRdujEabK%2F91C62p8zZ%2BGlaBN%2BDmNN2MvVUpveBafY%2BGFQq2Q8JUheg8mWELxdj%2FLZkxUEWYT%2FEvLbKn7P%2BN617TPpdOd%2BDcioHocyVjJW3KA3Br0sVQ1EpD8SyDo67CsZnH0a4KRjELpVrZvBEmPpqiCITXCM25Mn9CQtZcOWBJkKxquArIfer6Bx%2BKXQ9W6cJaFCI6l%2Ft0Ie5DtcWL225xSfTbDd5OuBZoiSdwn0a4swZ4Csm%2BdhwS6EmOOI8%2BhiEgwYvsl8sHslQ%2BPri9v53%2B%2BCEh5EP%2F1umO3duHq%2BG5%2B352alWwswQ%2BGpops7GTZ6NhKLxqdWw0T2yUNdIsykajVmxsVsnGYfebc2xU1GUqmtrBqWidqChrxCpKxWatqGhVScXL7nBwfXRkNDJkVOxSv5iMrRMZZY3YRclo14qMdpVk%2FH7njrvHxkUzw0Xj0Fw0T2%2BMGY20inKxXSsuqo7ISuPiza3bd0ajY2NjM8NG8%2BBs3Heu%2BVuysV2UjZ1asbFdJRu7d4Or8bFx0cpwsXlwLp5ObzIa6RTkolmvs9ROlVx07sfOcOAMjo2OdoaO1qHpKN5aK9rgdAeDq%2BHFsaHYyqBoHxzFvFP9hBvVzlpWypOKqvd4Uv0NP%2BqBhb8JSu4GS7aFj7tY3SjoY%2FV6%2BVgR5axksTzT9NZnF8xyYGZ3uoE4oEqD%2BOvArlc0RDz2vmyMhQ%2Fm7HKyxOG6h8HkmU3wLScppwCE4BGGN2gRkABFVBbCJzYGc3%2FBhNpRpnoWeB67fQ%2BEwZQJcOLPNh26XL5puBvnBc8UiLdZJTjiTeLGvtiklvfDulmZIy73IN2dMxRAKMZ4xKLmEgIP4kW%2BougyS1VMZDORceP5O4qUHmEIEwoco%2BtuS1DZZUGXXtwIOrINqN5wbIUNVGcC5b6kJhuqRq3TfnyEg1%2F0aYAYMGsRhK1NOc%2BTM6A3fc%2Bmx4LOJYim43jNKy%2FzSLKkpsKS1PljWmUhnUp356qEld%2FYkViWLq8mVkdkoqXXE1PhTMRSXb4BqDYF1eVInNBPZ48WRr8y%2BleaIaMIy5%2FwTyfXHB7%2FL44En%2BBPh5APD3%2Bl4Q5V8PFkAOn3iMMbQKVn7Pl41wn%2B9JnuweE3K939K0MsJwvYWkANNoBmuRuAzVlCzx38qOVJQgk4sg%2BIWvKbvKUIsynf5KuLepebR7QB8tx1xzU9FCoFSvvMkKG0Fal9XwulnU%2Bz%2FUysrZGKwLDJFgrCbMJr6W4fSluQIjSyT%2F25nM1HfJLia07xGauWFOJgRBhEz8ycOCCfDN4UzUzS65UnaOf36n%2FMCf6TiviyG5%2B2Ja%2Fbt9%2B64yt3uIefRULflOkqRA%2BUgdvOfLXZVMViVIvnBw7iaXH76XRcl%2Fo%2B3XT%2BBw%3D%3D"></iframe>
</div>

## Object Header

Objelerin structure'ına bakarken ilk öğrenmemiz gereken kısım objenin başlığıdır. Bu başlık; objeyi etkileyen önemli flagler, objede kullanılacak sub-header'lar, objenin tipi vb. gibi önemli bilgileri barındırır. Burada x64 sistemdeki object header yapısını görebilirsiniz:

```cpp
lkd> dt nt!_OBJECT_HEADER
   +0x000 PointerCount     : Int8B
   +0x008 HandleCount      : Int8B
   +0x008 NextToFree       : Ptr64 Void
   +0x010 Lock             : _EX_PUSH_LOCK
   +0x018 TypeIndex        : UChar
   +0x019 TraceFlags       : UChar
   +0x019 DbgRefTrace      : Pos 0, 1 Bit
   +0x019 DbgTracePermanent : Pos 1, 1 Bit
   +0x01a InfoMask         : UChar
   +0x01b Flags            : UChar
   +0x01b NewObject        : Pos 0, 1 Bit
   +0x01b KernelObject     : Pos 1, 1 Bit
   +0x01b KernelOnlyAccess : Pos 2, 1 Bit
   +0x01b ExclusiveObject  : Pos 3, 1 Bit
   +0x01b PermanentObject  : Pos 4, 1 Bit
   +0x01b DefaultSecurityQuota : Pos 5, 1 Bit
   +0x01b SingleHandleEntry : Pos 6, 1 Bit
   +0x01b DeletedInline    : Pos 7, 1 Bit
   +0x01c Reserved         : Uint4B
   +0x020 ObjectCreateInfo : Ptr64 _OBJECT_CREATE_INFORMATION
   +0x020 QuotaBlockCharged : Ptr64 Void
   +0x028 SecurityDescriptor : Ptr64 Void
   +0x030 Body             : _QUAD
```

Bu yazı windows 10 x64 işletim sistemi versiyonuna göre yazıldığı için, bu versiyonun object header structure'ı aşağıdaki gibidir: 

```cpp
//0x38 bytes (sizeof)
struct _OBJECT_HEADER
{
    LONGLONG PointerCount;                                                  //0x0
    union
    {
        LONGLONG HandleCount;                                               //0x8
        VOID* NextToFree;                                                   //0x8
    };
    struct _EX_PUSH_LOCK Lock;                                              //0x10
    UCHAR TypeIndex;                                                        //0x18
    union
    {
        UCHAR TraceFlags;                                                   //0x19
        struct
        {
            UCHAR DbgRefTrace:1;                                            //0x19
            UCHAR DbgTracePermanent:1;                                      //0x19
        };
    };
    UCHAR InfoMask;                                                         //0x1a
    union
    {
        UCHAR Flags;                                                        //0x1b
        struct
        {
            UCHAR NewObject:1;                                              //0x1b
            UCHAR KernelObject:1;                                           //0x1b
            UCHAR KernelOnlyAccess:1;                                       //0x1b
            UCHAR ExclusiveObject:1;                                        //0x1b
            UCHAR PermanentObject:1;                                        //0x1b
            UCHAR DefaultSecurityQuota:1;                                   //0x1b
            UCHAR SingleHandleEntry:1;                                      //0x1b
            UCHAR DeletedInline:1;                                          //0x1b
        };
    };
    ULONG Reserved;                                                         //0x1c
    union
    {
        struct _OBJECT_CREATE_INFORMATION* ObjectCreateInfo;                //0x20
        VOID* QuotaBlockCharged;                                            //0x20
    };
    VOID* SecurityDescriptor;                                               //0x28
    struct _QUAD Body;                                                      //0x30
}; 
```

Object header, Windows x86 mimarilerde 0x18 byte, x64 mimarilerde ise 0x30 byte'tır. Bu detay objenin header kısmını ayrıştırırken veya header'ı/objeyi ilgilendiren *Ob* fonksiyonlarını anlamada işimize yarayabilir. 

```cpp
lkd> ?? sizeof(nt!_OBJECT_HEADER)
unsigned int64 0x38
```

`PointerCount`, Windows'taki nesnelerin geçerli işaretçi sayısını takip etmek için kullanılan bir alanı ifade eder. Bu sayı, nesneye doğrudan veya dolaylı olarak yapılan referansların toplam sayısını ifade eder. Yani nesneye yapılan tüm belirteçlerin sayısı, PointerCount değerini oluşturur.

`HandleCount`, bir nesneye atanmış olan ve halen kullanımda olan handle sayısını belirtir. Bu, nesnenin aktif handle sayısını ifade eder. Handle, bir nesneye erişmek için kullanılan bir numaralı referanstır. HandleCount, bir nesneye açık handle sayısını kaydeder ve silinen (ancak hala bekleyen) handle'ları da hesaba katar. Bir nesnenin bellekten silinip silinmeyeceğini belirlemek için HandleCount alanı kullanılır. HandleCount, açık handle sayısını takip eder ve 0 olunca nesne bellekten silinir. Bu nedenle, hangi nesnenin bellekten silineceği, HandleCount'un 0 olması ile belirlenir.

Her iki alan da, bir nesneye olan referansları takip etmek için kullanılır. Ancak, PointerCount, nesneye olan toplam referans sayısını kaydederken, HandleCount yalnızca kullanımda olan handle'ların sayısını kaydeder. Kısacası PointerCount nesneye olan açık referansları ve HandleCount kullanımda olan açık handle'ları sayar. Bu nedenle iki alan arasındaki fark nesne için henüz silinmemiş olan ancak açık olmayan referansları temsil eder.

```cpp
lkd> dx (nt!_OBJECT_HEADER*)0xffffab8443d3e080
(nt!_OBJECT_HEADER*)0xffffab8443d3e080                 : 0xffffab8443d3e080 [Type: _OBJECT_HEADER *]
    [+0x000] PointerCount     : 3 [Type: __int64]
    [+0x008] HandleCount      : -92890414718840 [Type: __int64]
    [+0x008] NextToFree       : 0xffffab8443d3e088 [Type: void *]
    [+0x010] Lock             [Type: _EX_PUSH_LOCK]
    [+0x018] TypeIndex        : 0x98 [Type: unsigned char]
    [+0x019] TraceFlags       : 0xe0 [Type: unsigned char]
    [+0x019 ( 0: 0)] DbgRefTrace      : 0x0 [Type: unsigned char]
    [+0x019 ( 1: 1)] DbgTracePermanent : 0x0 [Type: unsigned char]
    [+0x01a] InfoMask         : 0xd3 [Type: unsigned char]
    [+0x01b] Flags            : 0x43 [Type: unsigned char]
    [+0x01b ( 0: 0)] NewObject        : 0x1 [Type: unsigned char]
    [+0x01b ( 1: 1)] KernelObject     : 0x1 [Type: unsigned char]
    [+0x01b ( 2: 2)] KernelOnlyAccess : 0x0 [Type: unsigned char]
    [+0x01b ( 3: 3)] ExclusiveObject  : 0x0 [Type: unsigned char]
    [+0x01b ( 4: 4)] PermanentObject  : 0x0 [Type: unsigned char]
    [+0x01b ( 5: 5)] DefaultSecurityQuota : 0x0 [Type: unsigned char]
    [+0x01b ( 6: 6)] SingleHandleEntry : 0x1 [Type: unsigned char]
    [+0x01b ( 7: 7)] DeletedInline    : 0x0 [Type: unsigned char]
    [+0x01c] Reserved         : 0xffffab84 [Type: unsigned long]
    [+0x020] ObjectCreateInfo : 0xffffab8443d3e098 [Type: _OBJECT_CREATE_INFORMATION *]
    [+0x020] QuotaBlockCharged : 0xffffab8443d3e098 [Type: void *]
    [+0x028] SecurityDescriptor : 0x1d7bd0002 [Type: void *]
    [+0x030] Body             [Type: _QUAD]
    ObjectName       : ""
    ObjectType       : WmiGuid
```

Diğer önemli kısımlardan devam etmek gerekirse, `TypeIndex` obje tipini bulmak için kullanılan 8 bitlik bir değerdir. Bu tip, nesnenin body kısmını yönetecek executive sub-system'ı belirleyecektir. 

`InfoMask` bölümü ise hangi sub-header'ın kullanılacağını belirten bir bit maskesidir. Windows 10'da 8 adet optional header vardır. Bu bölüm ayrı bir başlıkta detaylı olarak yazının ilerisinde işlenmiştir.

Objenin özelliklerini temsil eden bölüm ise `Flags` bölümüdür. Object header'da dikkat ederseniz Flags bölümünün altında 8 adet char tipi bölüm vardır. 

```cpp
[+0x01b ( 0: 0)] NewObject        : 0x1 [Type: unsigned char]
[+0x01b ( 1: 1)] KernelObject     : 0x1 [Type: unsigned char]
[+0x01b ( 2: 2)] KernelOnlyAccess : 0x0 [Type: unsigned char]
[+0x01b ( 3: 3)] ExclusiveObject  : 0x0 [Type: unsigned char]
[+0x01b ( 4: 4)] PermanentObject  : 0x0 [Type: unsigned char]
[+0x01b ( 5: 5)] DefaultSecurityQuota : 0x0 [Type: unsigned char]
[+0x01b ( 6: 6)] SingleHandleEntry : 0x1 [Type: unsigned char]
[+0x01b ( 7: 7)] DeletedInline    : 0x0 [Type: unsigned char]
```

Bu alanlar aslında hangi flag'lerin açık olduğunu gösterir. Her birisi farklı bir anlam taşır, bu bölüm hakkında daha fazla bilgi yazısının ileriki kısımlarında verilmiştir.

`SecurityDescriptor` objeyi kimin kullanabileceğini ve objeyle neler yapabileceğine dait güvenlik özelliklerini belirler.

## Optional Header'lar

Windows'un eski sürümlerinde sırasıyla:

- OBJECT_HEADER_NAME_INFO

- OBJECT_HEADER_HANDLE_INFO

- OBJECT_HEADER_QUOTA_INFO

sub-header'lar varlığını belirtmek için OBJECT_HEADER old version'da olan

- NameInfoOffset

- HandleInfoOffset

- QuotaInfoOffset

bölümlerini kullanırdı. Bu xxxInfoOffset alanları, OBJECT_HEADER yapısının başlangıcından itibaren karşılık gelen OBJECT_HEADER_xxx_INFO yapısının ofsetini belirleyen bir sayı (0 ile 265 arasında) içerir. Bu alanlardan herhangi birinin değeri sıfırsa, object manager karşılık gelen başlığın var olmadığını varsayar. Varlığı xxxInfoOffset alanları yerine OBJECT_HEADER->Flag alanında bir bit tarafından kontrol edilen başka bir sub-header, yani OBJECT_HEADER_CREATOR_INFO vardır.

Object manager basitçe hangilerinin mevcut olduğunu öğrenmek için OBJECT_HEADER başlangıcından itibaren ilgili ofsetlerini hesaplamak için kullanabilir.

Aşağıdaki şekilde, sub-header'lar ve bunların ofsetlerinin yerleşimi gösterilmektedir:

<div>
<iframe width="100%" height="650" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7Vxbd6I6FP41PrYLEgF9VGuPnenYTts505mXrghROUXiCbHV8%2BtPgHDHCwLajrrWTM3OBdjfzpe9syMN2Jst%2F6JoPv1GDGw1gGQsG%2FCqAUCrCfj%2FrmDlC5qq7Asm1DR8UUzwaP6HhVAS0oVpYCfRkBFiMXOeFOrEtrHOEjJEKXlPNhsTK3nVOZrgjOBRR1ZW%2BtM02FQ8liJF8gE2J9PgyrIkakZIf51QsrDF9RoASt7Hr56hYCwhcKbIIO8xEew3YI8Swvxvs2UPW65qA7X5%2Fa7X1Ib3TbHNdunw8%2Flv4%2FfX0RDrN79%2Bgy8PEL%2BRC9AUN8dWgUK8R8JuJ6kBu4SyKZkQG1m3hMy5UObCfzBjKwElWjDCRVM2s0QtXprsOfb9lxjK%2FX61jBdWouAwSl5xj1iEevcAr70PrzGQM%2FXuxR2Ma3DuXtIy7VdefBdwuSP4T4GNDNaRdoTIIQuqi1ZvbNBG9tDs398%2Bqz%2Fe0OvV19mFHGLDTR6TGWZ0xftRbCFmviXHR8L4JmG7sOs9MfmVgSTmiSYrfhcxTUBbSg7BEJ1gJnpFMPIvsduIRB64RYBWDgO0fOpAq2oSaKhIl3L8Uxfs658RSG%2FIWuCAplSLK6o7Jt5d6yEW6r8Ll4q4huHY%2B8RF6sT9e9f90u89vQz6nav%2Bw8v3H3dPnZeb4fUdr0czbjNde%2BTM%2FfaW3%2B%2Fav4zffYMFuqi%2FT02GH%2BfIA%2B2dLzZJaxublhWzHMG1m2wKUV3YbmQ5OYbyhinDy5goaxOiFmjS5Zpp%2FB6tHVAWpDqNrRuqtN6QEqAXRlipFuI5M10SCMYY0aBmgJGBqZOt2BVtrmSWhDQJnU1snMJZiJBlTmxe1Dl0mMu7LmQmX8M7omJmGoZ7mVwbSvJcBWYgB6AHU7ytZIwgwDtuA7ACG8gl9zpn%2BaAzvLrtf7ZpngEzB%2FIC0xwqB5zm%2BRBLG9Racv0OlmnPUeNPvt8KzpVLV6E74BZi47nFaECvtGqUWdHznRy%2Fnb%2BmblJmPUs%2FSPl4CjisjyfX6MwXsZF9LWG9bR3DRkBJGyk33Wt0108PSnhMKIFWH5RVhNipYCsRiIWRVzKEkSUN1ABl7UG3kvbA6g66t%2FllkU0ETphbceF4uHd4A67Npe9phU5aMWcudNcDwSUfQfwLvbdRUVf%2B83t0oSkczaMD6sfmhV12ZC6kSwlyt6cJm1ILqpIMW1q1xFCWutds0oCkNTTlwxID3EYMJQK2YedbGK6dyGQO4TvaZG7WCGjvod95uns4NUy1Y2MKWhm1upT2KIoRPfcjabceCt87KK98Jz1XU8phSFrbjaQ7lKJVrNncbeBsuA5MXkeVlPhwW9sH9xXZmn8Hla4YW3d5t7qSMijpSg7RDN%2FYY3I3HjuYnQ4TKbvu8de2vwvljFrPTJSrKbUeJkrn9Fo1MZGWYiK4hYm0%2FPuqlYnU4zPRANmGdZJcpIKjcxH8c7noEkCgKC0ZthXYbLWaMqyWnbRa2Cm9ywV3TDsUZSetmbJFdTM7pdtDcAB20o7PTt95EZ0kOe2aJa2PnLIZsAOS016JlbDwgRIrIvDdmlhpVsFnRWlIToWFanszDaXbK9IBaKh1fBq6ttDEOSHyaR2bfOT2BrWec7a%2Bkto7UotcNpIrt%2FX3QU7b%2FBlQlnV7y0GZ3Ts5Q7k3lK1jQtkuvqhWkjjfK02%2FJtuee2w2k3%2FPtirhNuTklNa6Bfue7f38%2FoO2aw4xvfdXnf9Q7TFezw3GtMeRYfsD%2B5EObZdCXIaZMOAye2y7nXNsu1kb4KBSwP1d0TPea%2FCG7aPjXe2xjyGH6olcU4zPcGfhbu06vWG7Lri3ZkqKbyt2LaK%2F9qaun2acUQeylN4Lyi7iMjjoHN%2B6AV0I9EesL6jJVlfY0ak5Z4SeUc%2BgruWc5Tww6lv3%2Bwqh3iXG6owzkANXO8A550hYkGM6CMwgbwFPaT%2FILfJ5a626FOmvbkS9LSRKxk8WGmHrnjim%2B1tMLrPwmMXQuE1Vh6gE%2BFFfF%2BvhS0VgUWJ0y2ZDuZhLScKZ99vZvFlbI555OwopPLFtdNzXTLiQWshxTN3TE6IsK84FtJpNnKxWY1pTcrQWyCo%2BhJf5CYW%2Ft5RJLn%2B8I9fSGeuCx5z2xzr7DoTDYp0XWJ%2BxLvTTqF2xru%2FFJrwYvR3Hbx69gQj2%2Fwc%3D"></iframe>
</div>

> Yukarıdaki 4 optional header'ların boyutları sabit olduğundan ve her zaman belirli bir sırada saklandıklarından, bu yapılara ofsetleri depolamaya çok az ihtiyaç vardır. Bu nedenle, Windows 7'de, 3 xxInfoOffsets alanı, her bir sub-header için bir bit içeren ve varlığını gösteren tek bir OBJECT_HEADER->InfoMask alanıyla değiştirilmiştir.

Yukarıdaki yapı windows object header'larda old version için geçerliydi. New version için InfoMask adında bir bölüm getirilerek sub-header'ları belirtmek için bir bit maskeleme yöntemi kullanılmaya karar verildi. Ayrıca new version ile farklı sub-header'lar da eklendi. Tüm sub-header'ları ve bit flag'lerini görebilmek için aşağıdaki tabloya bakabilirsiniz:

| Bit  | Bitmask Header     | Optional Object Header          | Description                                                                                                                                                                                                                                               | Offset                               |
| ---- | ------------------ | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| 0x01 | ObjectCreatorInfo  | nt!_OBJECT_HEADER_CREATOR_INFO  | Aynı türdeki tüm nesnelerin listesini ve nesneyi oluşturan işlemi işaret eden bir işaretçi içerir                                                                                                                                                         | ObpInfoMaskToOffset[0])              |
| 0x02 | ObjectNameInfo     | nt!_OBJECT_HEADER_NAME_INFO     | Adlandırılmış bir nesne ise nesnenin adını ve nerede olduğunu belirten bir _OBJECT_DIRECTORY yapısı işaretçisi içerir                                                                                                                                     | ObpInfoMaskToOffset[InfoMask & 0x3]  |
| 0x04 | ObjectHandleInfo   | nt!_OBJECT_HEADER_HANDLE_INFO   | Nesneye açık bir handle’ı olan tüm process’leri track’lemek için kullanılır. Bu yüzden _OBJECT_HANDLE_COUNT_ENTRY yapısını veya bir _OBJECT_HANDLE_COUNT_DATABASE yapısı içerir, bu da _OBJECT_HANDLE_COUNT_ENTRY listesi ve toplam girdi sayısını içerir | ObpInfoMaskToOffset[InfoMask & 0x7]  |
| 0x08 | ObjectQuotaInfo    | nt!_OBJECT_HEADER_QUOTA_INFO    | Nesneye ayrılan bellek kaynaklarıyla ilgili bilgileri içerir.                                                                                                                                                                                             | ObpInfoMaskToOffset[InfoMask & 0xF]  |
| 0x10 | ObjectProcessInfo  | nt!_OBJECT_HEADER_PROCESS_INFO  | Nesnenin sahibinin yürütme işlem yapısı işaretçisine sahiptir.                                                                                                                                                                                            | ObpInfoMaskToOffset[InfoMask & 0x1F] |
| 0x20 | ObjectAuditInfo    | nt!_OBJECT_HEADER_AUDIT_INFO    | File objeleri için Audit yani denetim yapılacaksa bu header kullanılır.                                                                                                                                                                                   | ObpInfoMaskToOffset[InfoMask & 0x3F] |
| 0x40 | ObjectExtendedInfo | nt!_OBJECT_HEADER_EXTENDED_INFO | Object Footer'ının pointer'ını tutar. Yani bu header sadece objenin footer'ı varsa kullanılır.                                                                                                                                                            | ObpInfoMaskToOffset[InfoMask & 0x7F] |
| 0x80 | ObjectPaddingInfo  | nt!_OBJECT_HEADER_PADDING_INFO  | Process ve Thread objeleri bu bayrağı set etmelidir. Object Body kısmının bellekteki hizzalamasını ayarlar.                                                                                                                                               | ObpInfoMaskToOffset[InfoMask & 0xFF] |

Peki bu optional header'lardan hangileri açık? Ve açık olan header'ların bilgilerini nasıl görüntüleyebiliriz? Bunun için önce InfoMask alanına bakmalıyız. Bu alan hangi header'ların açık olduğunu bize gösterebilir. 

Aşağıda adım adım hangi optional header'ın objede olup olmadığını bulma gösterilmiştir:

1. Sub-header'ların tablosuna baktığınızda Bit sütununu göreceksiniz.

2. Tüm sub-header'ların bitleri ile InfoMask değeri AND işlemine sokulur.

3. Eğer sonuç False değilse (0 harici bir değer) ilgili objede AND işlemine sokulan bite sahip optional header açık demektir.

Örneğin aşağıdaki gibi tek tek AND işlemine sokularak açık olan sub-header'lar bulunmuştur:

```cpp
lkd> dt nt!_OBJECT_HEADER ffff948ed404c050 -y Inf*
   +0x01a InfoMask : 0x88 ''
lkd> ? 0x88 & 0x1
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x2
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x4
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x8
Evaluate expression: 8 = 00000000`00000008
lkd> ? 0x88 & 0x10
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x20
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x40
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0x88 & 0x80
Evaluate expression: 128 = 00000000`00000080
```

Görüldüğü gibi 8 ve 128 yani ObjectQuotaInfo ve ObjectPaddingInfo header'ı şu anda açık. Peki açık olan header'ları nasıl görüntüleyebilirim? Sub-header'ların bellekteki yerlerini alma sırasını ve işletim sistemi mimarisine göre boyutlarını bilmeliyiz. x64 sistemde optional header'ların boyutları aşağıdaki gibidir:

```cpp
lkd> ?? sizeof(nt!_OBJECT_HEADER_PADDING_INFO)
unsigned int64 4
lkd> ?? sizeof(nt!_OBJECT_HEADER_EXTENDED_INFO)
unsigned int64 0x10
lkd> ?? sizeof(nt!_OBJECT_HEADER_AUDIT_INFO)
unsigned int64 0x10
lkd> ?? sizeof(nt!_OBJECT_HEADER_PROCESS_INFO)
unsigned int64 0x10
lkd> ?? sizeof(nt!_OBJECT_HEADER_QUOTA_INFO)
unsigned int64 0x20
lkd> ?? sizeof(nt!_OBJECT_HEADER_HANDLE_INFO)
unsigned int64 0x10
lkd> ?? sizeof(nt!_OBJECT_HEADER_NAME_INFO)
unsigned int64 0x20
lkd> ?? sizeof(nt!_OBJECT_HEADER_CREATOR_INFO)
unsigned int64 0x20
lkd> ?? sizeof(nt!_OBJECT_HEADER)
unsigned int64 0x38
```

Açık olan sub-header'ın içerisindeki bilgileri görüntüleyebilmek için, offsetini hesaplamamız gerekiyor. Deminki AND işlemini yaptığımız örnekte ObjectQuotaInfo ve ObjectPaddingInfo header'larının açık olduğunu bulduk. Yukarıdaki boyutlara göre 0x20 ve 0x4 bizim offset hesaplayabilmemiz için gerekli verilerdir. Fakat ondan önce aşağıdaki diyagrama iyice bakın. Çünkü offset hesaplanırken bu diyagramda verilen sıra referans alınacaktır (bellek dağılımına göre çizilmiştir).

<div>
<iframe width="100%" height="900" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R5Ztdc6IwFIZ%2FjZedAQKUXqLQ2t1WXLXT9qpDMQpTFDeNFffXLx9BwcSp0xbiEC78OCeEcJ4cjm%2FADugt4hvkrvz7aArDjiJN4w6wOopiqErymhq2uUHV5dwwR8E0N5UM4%2BAfJEaJWNfBFL5XGuIoCnGwqhq9aLmEHq7YXISiTbXZLAqrR125c0gZxp4b0tbHYIp9clqatLf3YTD3iyPLEvG8ut7bHEXrJTleRwFStuXuhVv0RQzvvjuNNiUTsDugh6II558WcQ%2BGaWiLsOX7XR%2Fx7saN4BKfssMH7l%2B5y0FgD%2B%2Be9IcP9836vbggvXy44RoWp6GHSX%2FdWZR0m4Y9jFDm0f%2Bu06F2k%2FOcZVvZpM%2FT96Hj3L30bdOyR0U3yXjynvIWJBR4W4Q%2FCyBMhygn7o0fYDheuV7q3STzLbH5eBES9ywIw95uQEW4Qfcdo%2BgNljzX2ZZ4XOSRKSftjl4OWXH%2BEGEYl0wkhDcwWkCMtkkT4k0meL4Lme5AAvn3zX7yAJm08UsTRyXTwCXzdb7res8s%2BUCwsREOJb3%2Fqsib9VC%2Bf8YAweft5OKKQuh0f9m9CQHxMjQt63Zw83I7uHb4hj%2BdCKxWRxgwSJ2OReWNpbi%2BHeViP03sgWVbgoExuIOhr3lVMOaDdTsRi4qqcKeifHYZGzk9ezwWjIvOnQv4hMufB2diikVF417zi4MdpdI3B9adLRgW%2FjVf%2BwTLwLwXDQr%2Feq9TUApxstrLkq%2BLnirh3sg2J86IQD6igXbm1Tnpoh%2FlrvP%2FRXF5lPtXMJvISyF4eI2S3syiq1dUNMjieb6q91t0NVWv0FVkmq4hNQnX%2BFG4saELg06j0amNoqMXLL6FTldbi844M3QKvabxHXRSbLQW3UHWsZYJm0X3syu9Upyub7WUnXFm7ABd7KRY5RvnGiUElTsMXdds%2FOmKlcVfai%2BCwxTgjUClK09WPVqM4DALGEK6WQR0BcmrQIsZHKYBdwaK6GnAulPRLAJ6QVy0NODPgF7%2BFi0NGDeGmkVAL3ULlwbcGdAr261ncJAHrFtxzTKgV5mlWGk3A%2BPcGLD0sdHe%2BB%2FmAHdxxtLHskAJwBtA8bCqyIWAtzTTWPJYsELAnQFLHouVB6z77s0yYOljsfKAPwNaH9f3yIvQT7mc%2BnymrtWFmtbhdaDuOtazyKAvFeM00LJeW1Iz1b44MoeVa6yran2pxpL6QKCyVh%2BA5Ov%2Bj3CZr%2FRnQ2D%2FBw%3D%3D"></iframe>
</div>

Object Header'a en yakın ObjectQuotaInfo'nun boyutu 0x4 idi. O zaman aşağıdaki gibi Object Header'ın adresinden 0x4 çıkarıp ObjectQuotaInfo başlığını aşağıdaki gibi görüntüleyebiliriz:

```cpp
lkd> dt nt!_OBJECT_HEADER_QUOTA_INFO ffff948ed404c050-0x20
   +0x000 PagedPoolCharge  : 0x1000
   +0x004 NonPagedPoolCharge : 0xc48
   +0x008 SecurityDescriptorCharge : 0x78
   +0x00c Reserved1        : 0
   +0x010 SecurityDescriptorQuotaBlock : 0xffff948e`c4cb1d40 Void
   +0x018 Reserved2        : 0
```

İkinci açık olan header ise ObjectPaddingInfo'dur ve header boyutu 0x20 dir. Fakat bu başlıktan önce ObjectQuotaInfo başlığı açık olduğundan bu başlığın da değerini (0x4) üzerine eklenip offset bulunur. Aşağıdaki gibi ObjectPaddingInfo başlığının offsetini hesaplayıp görüntüleyebiliriz:

```cpp
lkd> dt nt!_OBJECT_HEADER_PADDING_INFO ffff948ed404c050-0x20-0x4
   +0x000 PaddingAmount    : 0x20
```

Bu uygulamanın daha iyi anlaşılması adına yapılan örneği temsil eden diyagram aşağıdaki gibidir:

<div>
<iframe width="100%" height="900" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7V1bc5s4FP41mWkfkgFx9aOvTXaT2E3c2eSpIxvZpsHgxTix99evAAkDEo4vXFxTT6c2RyDE%2Bc4nHR0dlCupPV9%2Fc%2BFi9uAYyLoCgrG%2BkjpXAIj4H%2F7yJZtQoqqNUDB1TYOctBU8m%2F8hIhSIdGUaaJk40XMcyzMXSeHYsW009hIy6LrOR%2FK0iWMl77qAU8QInsfQYqX%2FmIY3C6W6Imzlt8iczuidRYGUjOD4beo6K5vc7wpIQvAJi%2BeQ1kUEyxk0nI%2BYSOpeSW3Xcbzw13zdRpavW6q28LpeRmnUbhfZ3j4XvHu3DWg%2Fmt3B%2FYv64x2%2Bdf6eX5Na3qG1QvQxVAvX15o4uFpf7ZbjBiXqvyu%2FqS38nJPgExepU%2F970O%2Ff%2F7ztNjvdJ1oNbk9YU3gGUYW3oeoPFIj8Joq4%2BGNmeuh5Acd%2B6Qc2OCybeXOLFE9My2pHDaLqllpLz3XeUKykF3xwCXTHxOSE6O5xldHnR66H1jERUeE35MyR527wKaQUADm8hJi7JEjh8cfWeCSRnDOLGY5MzAASe51GVW8xwz8IbHwIB4J6OwLix2ogPrx6koteN8NrXdqhVP%2BpHdebOVPHhta94yyIKn8hz9sQ1cCV5yQVjdam90Iu93%2B%2FxuSddaygsyEHDAQdkYIzgwv%2FLpZpv%2FkQEz35RQZcziLsw6dABkNXBq2ls3LHaIdKGnwIXWRBz3xP1s%2FDg1w6cMyAA1FfJyShp5qnVXjQnSKPXJVCNWrG8UA3Mrm6XEA7YQGUmNs%2B6poQuenrYTr6gh8Ft0KgX19DFqf43G%2F91W0PCaN%2FDpqdzt3jt593j71%2BjN7hzQug99aC9qT31ILLJbVH2tsKpOJn0ihqc0EHx6szo0fgWOL%2BnYRcdSdBR9uY8STB7b4Mu4%2BdboegW2UvXSYweuXAsCNwEpjmj87dsF6oyKByVMAnqAye%2Bu3u83PNcFGrxkUHZ%2BnriFqvB7qsr5Pwb%2BKOT46%2BjigV4%2ByoSWdHkct1dkTpEwZ%2B%2F9EfNovgX6%2BHm3w6%2F7g%2BhwVHyGpFThmP9wXwVql8jkJvlonmbfOxc9%2BtV3eqVO8VKp%2FA8th8qBso1XuEKgMKnW0ttvOs44M0SYTbT93msP%2BUntQlYzaReHFOcZxccVer9zm1TNyPgbnpjn0Qxt7KRcG0P6xq5NITAn2eb5TuJHQVWU2gC0QWXV0oE1w9V3DXulob6BQWOrlU6LLjbkdBp8oXC51%2BZtABNup1CnTCWr9Y6FKs4y1rlAtdvitTwtqPgF4odvqZYSexg52wlqvVc4FTCIY7nHldufpnR6xA%2F8LlQpCmQNUQyOzIE4weFwxBmgWciXS5ELAjSDgKXDAGaRpUjgGoOw14a1nlQsAG0utGg%2BoxYMPfdaMBZ%2BmwXAjYUHftaFA5Bmxk%2B%2BIxSPGAtxRXLgZslFlYg%2FPFIFwxdVwDubTYdmxUDEOqR4c3c9bPE5ki2FH5tI03cxbPlBpFEKBqAGjafZ2HiKonbQpv4nzGQ0QRPKgcA97EuV484K3Il4sBb%2BZcLx5UjoGW1RelAMBVmYslOl75WUrNzvXkvLxyou61GyXZCwkylcSzUjRW%2F1SWv%2F5LerkoyrHdL%2BE2ytJMgJD1ptGNKMoNGShA1yRFlQDQIsRyScFVZD6wOafgqmq5KbgKG7UqLhGt1rln%2B%2BbVq0pRQx0bHSsC6la%2F81pnoDWg7wd0RP38kebG4OoTYuBxjefRFEc1XgBOqpFLWTUAGm9qxWanlOdRttvkiuI9Skmu3qPclQhU2itcWIPu5oWe5h%2B8xku2FwVHST%2BUbhMh7yRYhjN6DW50RRNlEP2fszNKXx8IncNdJ57qtZ5mBkpxZiAebgbCQWaQhHaHDTD28scMUmag%2Fu69wR7c32EG0o2U%2FOQ8MwV7WoF86pYZp1mBVu%2FOIG%2FUpX1R1ytFfdesowaonzQE5B5tUmjWdnotnFYRmh0TbWIqytxjo6Swldb43YeUCh3M3M0q8130Q80qbZ9lm5W%2Ba4peg97qkDEqdytSQUaizsFWlJWlXpYVicVZ0aX7u%2FkPeVnJeYdaVdo8S7eqAmIq2Rv6Fb%2Fmlv9SGjhuKe3wNbkMk8kN6QLCJgctcVfhQghJne%2B7FdGnFanpVZaiwSsg2LEnTatjnpzUuUSVcDDz5IrBO24HB1GbTABiV0SvwzSW%2Fd5I9tDa2wU7yYiOr0QQEbTMqY0PxxhmhOUtf0HBHEOrSQrmpmFYWascSeskz3%2FKigSzqxhvtx3OYpBU1HKoftzmDn9QTaCa7qB5%2B42ViWrjuH0fjLDr5KG6954dFwOqlhEOimEKSsU0e0OImZhHjsqdPXEe4PIt3CNZWOvZm31EYvbOuaTL4JsL4Z5NcO6Da4%2BW%2Fld%2F9AuNvQE0DNOe%2Bs3F9X5ZBp5B2GT5K7vN00H5Nvk%2BQ%2BYjfMenQt4DAOHrQe3NjXxLzCasVCxQtkfDwAfzd5Hhc87BBJtYwS7TM8xNZBfBQ6All%2FDFhsoSke5XG2di4%2FAVfHy4%2FXsEoeO0%2FaMPUvd%2F"></iframe>
</div>

Optional header'ların offsetini hesaplamak bu yöntemle görüldüğü gibi uzun bir zaman alıyor. Daha kolay ikinci yol; tüm sub-header'ların offset yerleşimlerini tutan bir pointer array vardır ve bu array'e `ObpInfoMaskToOffset` denir. Bu array, object header'ınızın versiyonuna göre değişiklik gösterir.

> Bu ofsetler, optional header'ların InfoMask bit dağılımı kombinasyonlarının tümü için mevcuttur

Toplamda 8 adet sub-header olduğuna göre pointer array boyutunun 2^8 olması gerekiyor, yani 256 byte'lık bir hafızayı `ObInitSystem` içerisinde allocate eder.

```cpp
ULONG i = 0;
ULONG offset = 0;
BYTE ObpInfoMaskToOffset[256];

do
{
    offset = 0;
    if ( i & OB_INFOMASK_CREATOR_INFO )
        offset = sizeof(_OBJECT_HEADER_CREATOR_INFO);
    if ( i & OB_INFOMASK_NAME )
        offset += sizeof(_OBJECT_HEADER_NAME_INFO);
    if ( i & OB_INFOMASK_HANDLE )
        offset += sizeof(_OBJECT_HEADER_HANDLE_INFO);
    if ( i & OB_INFOMASK_QUOTA )
        offset += sizeof(_OBJECT_HEADER_QUOTA_INFO);
    if ( i & OB_INFOMASK_PROCESS_INFO )
        offset += sizeof(_OBJECT_HEADER_PROCESS_INFO);
    if ( i & OB_INFOMASK_AUDIT_INFO )
        offset += sizeof(_OBJECT_HEADER_AUDIT_INFO);
    if ( i & OB_INFOMASK_EXTEND_INFO )
        offset += sizeof(_OBJECT_HEADER_EXTEND_INFO);
    if ( i & OB_INFOMASK_PADDING_INFO )
        offset += sizeof(_OBJECT_HEADER_PADDING_INFO);

    ObpInfoMaskToOffset[i++] = offset; // anlık hesaplanan toplam offset 

} while(i<256);
```

Şimdi yukarıdaki kodu doğrulamak adına ObpInfoMaskToOffset'i görüntüleyelim:

```cpp
lkd> db nt!ObpInfoMaskToOffset l100
fffff800`0aa25e60  00 20 20 40 10 30 30 50-20 40 40 60 30 50 50 70  .  @.00P @@`0PPp
fffff800`0aa25e70  10 30 30 50 20 40 40 60-30 50 50 70 40 60 60 80  .00P @@`0PPp@``.
fffff800`0aa25e80  10 30 30 50 20 40 40 60-30 50 50 70 40 60 60 80  .00P @@`0PPp@``.
fffff800`0aa25e90  20 40 40 60 30 50 50 70-40 60 60 80 50 70 70 90   @@`0PPp@``.Ppp.
fffff800`0aa25ea0  10 30 30 50 20 40 40 60-30 50 50 70 40 60 60 80  .00P @@`0PPp@``.
fffff800`0aa25eb0  20 40 40 60 30 50 50 70-40 60 60 80 50 70 70 90   @@`0PPp@``.Ppp.
fffff800`0aa25ec0  20 40 40 60 30 50 50 70-40 60 60 80 50 70 70 90   @@`0PPp@``.Ppp.
fffff800`0aa25ed0  30 50 50 70 40 60 60 80-50 70 70 90 60 80 80 a0  0PPp@``.Ppp.`...
fffff800`0aa25ee0  04 24 24 44 14 34 34 54-24 44 44 64 34 54 54 74  .$$D.44T$DDd4TTt
fffff800`0aa25ef0  14 34 34 54 24 44 44 64-34 54 54 74 44 64 64 84  .44T$DDd4TTtDdd.
fffff800`0aa25f00  14 34 34 54 24 44 44 64-34 54 54 74 44 64 64 84  .44T$DDd4TTtDdd.
fffff800`0aa25f10  24 44 44 64 34 54 54 74-44 64 64 84 54 74 74 94  $DDd4TTtDdd.Ttt.
fffff800`0aa25f20  14 34 34 54 24 44 44 64-34 54 54 74 44 64 64 84  .44T$DDd4TTtDdd.
fffff800`0aa25f30  24 44 44 64 34 54 54 74-44 64 64 84 54 74 74 94  $DDd4TTtDdd.Ttt.
fffff800`0aa25f40  24 44 44 64 34 54 54 74-44 64 64 84 54 74 74 94  $DDd4TTtDdd.Ttt.
fffff800`0aa25f50  34 54 54 74 44 64 64 84-54 74 74 94 64 84 84 a4  4TTtDdd.Ttt.d...
```

sonuncu index tüm header'ların açık olduğu offset değeridir. Bu da demek oluyorki 0x100 yani decimal olarak 255 değeri InfoMask olmasıdır. O halde verilen koda bakılırsa tüm header'ların toplam boyutu da son index değeri yani a4 olması gerekiyor. Hızlıca bir hesap yapmak gerekirse:

```cpp
lkd> ? 4 + 10 + 10 + 10 + 20 + 10 + 20 + 20
Evaluate expression: 164 = 00000000`000000a4
```

Peki  nasıl kullanılır bu ObpInfoMaskToOffset? Tam olarak aşağıdaki gibi bir mantıkla kullanılır

> Offset = ObpInfoMaskToOffset[OBJECT_HEADER->InfoMask & (DesiredHeaderBit \| (DesiredHeaderBit-1))]

En son manuel olarak hespaladığımız offset değerini padding header için 24, quota header için ise 20 olarak bulduk. Şimdi bu offsetleri kolay yoldan bulmak için şöyle bir yol izlenebilir:

```cpp
lkd> ?? ((unsigned char *)@@masm(nt!ObpInfoMaskToOffset))[0x88 & (0x8 | (0x8-1))]
unsigned char 0x20 ' '
lkd> ?? ((unsigned char *)@@masm(nt!ObpInfoMaskToOffset))[0x88 & (0x80 | (0x80-1))]
unsigned char 0x24 '$'
```

Fakat her türlü InfoMask değerinden hangi optional header'ların set edildiğini bulmak gereklidir.

Son olarak bilinmesi gerekilen iki farklı 

## Object Footer

Obje gövdesinin(body) sonunda ayrılır ve ancak `_OBJECT_HEADER_EXTEND_INFO` set edilmişse footer objeye dahil olur. Neden mi? Çünkü zaten ExtendedUserInfo, footer'ın pointer'ını tutar. Aşağıya x64 Extended sub-header c++ structure'ını bırakıyorum.

```cpp
//0x10 bytes (sizeof)
struct _OBJECT_HEADER_EXTENDED_INFO
{
    struct _OBJECT_FOOTER* Footer;                                          //0x0
    ULONGLONG Reserved;                                                     //0x8
}; 
```

Object footer internalde `_OBJECT_FOOTER` olarak geçer ve iki bölümden oluşur:

1. HandleRevocationInfo: File ve Key objeleri bu footer ile oluşturulur. Objenin ObCreateObjectEx fonksiyonu ile oluşturulması şarttır.

2. ExtendedUserInfo: Silo Context objeleri bu footer ile oluşturulur. Objenin ObCreateObjectEx fonksiyonu ile oluşturulması şarttır.

Örnek olarak random bir File objesinin adresini bulalım:

![random_file_object_find_01](/assets/img/2023-03-16-derinlemesine-windows-objects-ve-handles-0x02/random_file_object_find_01.png)

Daha sonra bu objenin header'ındaki InfoMask'i hesaplamak için görüntüleyelim:

```cpp
lkd> dt nt!_OBJECT_HEADER 0xffff948ed285c9b0-30
   +0x000 PointerCount     : 0n32767
   +0x008 HandleCount      : 0n1
   +0x008 NextToFree       : 0x00000000`00000001 Void
   +0x010 Lock             : _EX_PUSH_LOCK
   +0x018 TypeIndex        : 0x68 'h'
   +0x019 TraceFlags       : 0x1 ''
   +0x019 DbgRefTrace      : 0y1
   +0x019 DbgTracePermanent : 0y0
   +0x01a InfoMask         : 0x4c 'L'
   +0x01b Flags            : 0 ''
   +0x01b NewObject        : 0y0
   +0x01b KernelObject     : 0y0
   +0x01b KernelOnlyAccess : 0y0
   +0x01b ExclusiveObject  : 0y0
   +0x01b PermanentObject  : 0y0
   +0x01b DefaultSecurityQuota : 0y0
   +0x01b SingleHandleEntry : 0y0
   +0x01b DeletedInline    : 0y0
   +0x01c Reserved         : 0
   +0x020 ObjectCreateInfo : 0xffff948e`c4cb1d40 _OBJECT_CREATE_INFORMATION
   +0x020 QuotaBlockCharged : 0xffff948e`c4cb1d40 Void
   +0x028 SecurityDescriptor : (null)
   +0x030 Body             : _QUAD
```

InfoMask 0x4c = 0100 1100 değerindedir ve ObjectExtendedInfo 0x40 = 0100 0000 değerindedir. Görüldüğü gibi bu objenin footer'ı var ve geriye kalan sadece _OBJECT_HEADER_EXTEND_INFO yapısının offsetini hesaplayıp görüntülemek olacak:

```cpp
lkd> ?? ((unsigned char *)@@masm(nt!ObpInfoMaskToOffset))[0x4c & (0x40 | (0x40-1))]
unsigned char 0x40 '@'
lkd> dt nt!_OBJECT_HEADER_EXTENDED_INFO 0xffff948ed285c9b0-30-40
   +0x000 Footer           : 0xffff948e`d285ca88 _OBJECT_FOOTER
   +0x008 Reserved         : 0x4e564441`00000134
```

Footer bölümündeki pointer'ı da _OBJECT_FOOTER structure'ına uyarladığımızda objenin footer'ına ulaşmış olacağız:

```cpp
lkd> dt nt!_OBJECT_FOOTER 0xffff948e`d285ca88
   +0x000 HandleRevocationInfo : _HANDLE_REVOCATION_INFO
   +0x020 ExtendedUserInfo : _OB_EXTENDED_USER_INFO
```

Hatırlarsanız File ve Key objeleri HandleRevocationInfo ile oluşuyordu. Son adımda objenin footer'ı aşağıdaki gibidir:

```cpp
lkd> dx ((nt!_OBJECT_FOOTER*)0xffff948e`d285ca88)->HandleRevocationInfo
((nt!_OBJECT_FOOTER*)0xffff948e`d285ca88)->HandleRevocationInfo                 [Type: _HANDLE_REVOCATION_INFO]
    [+0x000] ListEntry        [Type: _LIST_ENTRY]
    [+0x010] RevocationBlock  : 0x0 [Type: _OB_HANDLE_REVOCATION_BLOCK *]
    [+0x018] AllowHandleRevocation : 0x1 [Type: unsigned char]
    [+0x019] Padding1         [Type: unsigned char [3]]
    [+0x01c] Padding2         [Type: unsigned char [4]]
```

## Objelerin Flag'leri

Bir objeyi oluşturma sırasında ve/veya objeye erişildiğinde davranışını değiştirmek için kullanılan başka bir bit maskeleme yöntemi kullanır. Bu alan, obje oluşturulurken _OBJECT_ATTRIBUTES yapısının Attributes alanından bazı verilerle doldurulur ve bu veriler Object Manager'a iletilir. Bu alana gelebilecek değerler aşağıdaki gibi tanımlanmıştır:

```cpp
#define OB_FLAG_NEW_OBJECT              0x01
#define OB_FLAG_KERNEL_OBJECT           0x02
#define OB_FLAG_CREATOR_INFO            0x04
#define OB_FLAG_EXCLUSIVE_OBJECT        0x08
#define OB_FLAG_PERMANENT_OBJECT        0x10
#define OB_FLAG_DEFAULT_SECURITY_QUOTA  0x20
#define OB_FLAG_SINGLE_HANDLE_ENTRY     0x40
#define OB_FLAG_DELETED_INLINE          0x80
```

> Bununla beraber 3 adet daha flag vardır fakat sadece runtime'da kullanılır. Yani bu bayraklar diğer bayraklar gibi store edilmiyor. OBJ_CASE_INSENSITIVE, OBJ_OPENIF, OBJ_OPENLINK.

| Bitmask Flag         | Object Manager Flag                   | Açıklama                                                                                                                                           |
| -------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| NewObject            | OB_FLAG_NEW_OBJECT (0x01)             | Objenin yaratıldığını ancak objenin namespace’e henüz yerleştirilmediğini belirler.                                                                |
| KernelObject         | OB_FLAG_KERNEL_OBJECT (0x02)          | Objenin yalnızca kernel handle'larına sahip olabileceğini belirler.                                                                                |
| KernelOnlyAccess     | OB_FLAG_CREATOR_INFO (0x04)           | Objenin yalnızca çekirdek modunda (kernel’in) handle açılabileceğini belirler.                                                                     |
| ExclusiveObject      | OB_FLAG_EXCLUSIVE_OBJECT (0x08)       | Objenin yalnızca oluşturan process tarafından erişilebileceğini ve kullanılabileceğini belirler.                                                   |
| PermanentObject      | OB_FLAG_PERMANENT_OBJECT (0x010)      | Bir objenin handle sayısı 0'a düştüğünde bile objenin yok edilmeyeceğini belirler.                                                                 |
| DefaultSecurityQuota | OB_FLAG_DEFAULT_SECURITY_QUOTA (0x20) | Security Descriptor’ın default olarak 2 KB kotasını kullanacağını belirler.                                                                        |
| SingleHandleEntry    | OB_FLAG_SINGLE_HANDLE_ENTRY (0x40)    | Opsiyonel OBJECT_HEADER_HANDLE_INFO yapısının sadece bir giriş içerdiğini ve handle listesi değil, yalnızca tek bir handle olduğunu belirtir.      |
| DeletedInline        | OB_FLAG_DELETED_INLINE (0x80)         | Objenin IRQL 0'daki işçi thread tarafından silinmek üzere kuyrukta olduğunu belirtir. Bu, dead-lock ve sistem çökmelerini önlemek için kullanılır. |

Hangi flag'lerin açık olduğunu bulabilmek için `_OBJECT_HEADER->Flags & Flag Biti` yapılır ve sonuç 0'ın harici bir değerse, seçilen Flag Biti objede set edilmiş demektir.

```cpp
+0x01b Flags            : 0xcf ''
+0x01b NewObject        : 0y1
+0x01b KernelObject     : 0y1
+0x01b KernelOnlyAccess : 0y1
+0x01b ExclusiveObject  : 0y1
+0x01b PermanentObject  : 0y0
+0x01b DefaultSecurityQuota : 0y0
+0x01b SingleHandleEntry : 0y1
+0x01b DeletedInline    : 0y1
```

```cpp
lkd> ? 0xcf & 0x01
Evaluate expression: 1 = 00000000`00000001
lkd> ? 0xcf & 0x02
Evaluate expression: 2 = 00000000`00000002
lkd> ? 0xcf & 0x04
Evaluate expression: 4 = 00000000`00000004
lkd> ? 0xcf & 0x08
Evaluate expression: 8 = 00000000`00000008
lkd> ? 0xcf & 0x10
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0xcf & 0x20
Evaluate expression: 0 = 00000000`00000000
lkd> ? 0xcf & 0x40
Evaluate expression: 64 = 00000000`00000040
lkd> ? 0xcf & 0x80
Evaluate expression: 128 = 00000000`00000080
```

Tabiiki de bununla uğraşmayacağız, yukarı kısımlarda bahsedildiği gibi object header'daki bölümlere bakarak anlayabiliriz :)

## Objelerin Tipi

Windows objeler için object header'ın "old version"" ve "new versiyon" olmak üzere iki farklı sürümü vardır (dikkat ettiyseniz object header bölümünde de new version yazsını görmüşsünüzdür) Sistemin x86 veya x64 komut kümesinde olması fark etmeksizin:

- old version, windows sistemlerin 7'den önceki sürümlerini kapsıyor.

- new version, windows sistemlerin 7 ve sonraki sürümlerini kapsıyor.

Bu versiyonlar arası değişiklikler aşağıdaki gibidir:

```cpp
//Windows Vista | 2008 - SP2 x64
//0x38 bytes (sizeof)
struct _OBJECT_HEADER
{
    LONGLONG PointerCount;                                                  //0x0
    union
    {
        LONGLONG HandleCount;                                               //0x8
        VOID* NextToFree;                                                   //0x8
    };
    struct _OBJECT_TYPE* Type;                                              //0x10
    UCHAR NameInfoOffset;                                                   //0x18
    UCHAR HandleInfoOffset;                                                 //0x19
    UCHAR QuotaInfoOffset;                                                  //0x1a
    UCHAR Flags;                                                            //0x1b
    union
    {
        struct _OBJECT_CREATE_INFORMATION* ObjectCreateInfo;                //0x20
        VOID* QuotaBlockCharged;                                            //0x20
    };
    VOID* SecurityDescriptor;                                               //0x28
    struct _QUAD Body;                                                      //0x30
}; 
```

```cpp
kd> dt nt!_OBJECT_HEADER
   +0x000 PointerCount     : Int4B
   +0x004 HandleCount      : Int4B
   +0x004 NextToFree       : Ptr32 Void
   +0x008 Type             : Ptr32 _OBJECT_TYPE
   +0x00c NameInfoOffset   : UChar
   +0x00d HandleInfoOffset : UChar
   +0x00e QuotaInfoOffset  : UChar
   +0x00f Flags            : UChar
   +0x010 ObjectCreateInfo : Ptr32 _OBJECT_CREATE_INFORMATION
   +0x010 QuotaBlockCharged : Ptr32 Void
   +0x014 SecurityDescriptor : Ptr32 Void
   +0x018 Body             : _QUAD
```

> Type, NameInfoOffset, HandleInfoOffset ve QuotaInfoOffset bölümleri new version ile kaldırılmıştır. Yerine Lock, TypeIndex, TraceFlags, InfoMask bölümleri getirilmiştir.

Farkı incelemek için yazının başında anlatılan OBJECT_HEADER ile kıyaslama yapabilirsiniz. 

Windows'ta index numaralarının tutulduğu bir pointer array vardır ve bu array'e `ObTypeIndexTable` denir. Bu array'deki her pointer (1. ve 2. pointer'lar istisnadır), o obje tipine özgü özellikler içeren bir OBJECT_TYPE veri yapısına işaret eder. Eğer eski versiyon kullanılıyorsa direkt olarak `_OBJECT_HEADER->Type` bölümünden obje tipi bulunabilirdi. Fakat Windows yeni versiyonunda obje tiplerini belirten bölümde XOR'lanmış bir index numarası saklar, yani TypeIndex bölümünde.

Windows her bir nesne türü için özel işlevler ve yöntemler içeren OBJECT_TYPE yapıları kullanır. Bu yapılar, ObTypeIndexTable'daki her TypeIndex numarasına karşılık gelir.

```cpp
lkd> !object ffff948e`d18e0340
Object: ffff948ed18e0340  Type: (ffff948eb86d17a0) Process
    ObjectHeader: ffff948ed18e0310 (new version)
    HandleCount: 6  PointerCount: 196557
lkd> dps nt!ObTypeIndexTable
fffff800`0aafce80  00000000`00000000
fffff800`0aafce88  ffffbc00`b6760000
fffff800`0aafce90  ffff948e`b86d1380
fffff800`0aafce98  ffff948e`b86d1900
fffff800`0aafcea0  ffff948e`b86d1e80
fffff800`0aafcea8  ffff948e`b86d1a60
fffff800`0aafceb0  ffff948e`b86d1640
fffff800`0aafceb8  ffff948e`b86d17a0
fffff800`0aafcec0  ffff948e`b86c5140
fffff800`0aafcec8  ffff948e`b86c5980
fffff800`0aafced0  ffff948e`b86c52a0
fffff800`0aafced8  ffff948e`b86c5da0
fffff800`0aafcee0  ffff948e`b86c4900
fffff800`0aafcee8  ffff948e`b86c56c0
fffff800`0aafcef0  ffff948e`b86c5400
fffff800`0aafcef8  ffff948e`b86c5ae0
```

Yukarıdaki çıktılara bakarsanız Type: (ffff948eb86d17a0) ObTypeIndexTable dizisinde bulunuyor. Obje tipinin çözülmesini şematize etmek gerekirse şöyle bir akış görebiliriz:

<div>
<iframe width="100%" height="700" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7V1bd9o4EP41nLP7EA62fH0MhDTdpk226bbbfckxIMCNQawRCeyvX8m2fJWxAdmmNZz2YMsX7Pm%2BGc2MRkoHDBbbd661mn9EE%2Bh05N5k2wE3HVmWyD%2FyRVt2foummX7DzLUnwUlRw5P9Hwwae0Hrxp7AdeJEjJCD7VWycYyWSzjGiTbLddFb8rQpcpK%2FurJmMNPwNLacbOs3e4Lnfquh9qL2O2jP5uyXpV5wZGSNX2Yu2iyD3%2BvIoOd9%2FMMLi90raFjPrQl6izWBYQcMXISwv7XYDqBDZcvE5l93m3M0fG4XLnGZC1bTqfFk3t32ZqsXID%2Bb3z9uf1zJmn%2BbV8vZQPYe3tPiHZMQuQ0Bg%2Bz03%2BY2hk8ra0yPvBE%2BkLY5XjhkTyKb1nrlIzS1t5D8an%2BNXfQCB8hBrncrcOt9yJGp7Tix9kBwoJ99qeA9X6GL4TbWFLzkO4gWELs7ckpwVDUDAgaE1IM3eovQlQy%2FaR4HNmizAj7NwhtHMiUbgVgPEbFULGKPRlRknhwLxNyIVBmLA6GaWkaogDE%2FIVVFOV2sXxV89TT5YOB%2Fhrfmf7fr%2FssKXPGkqjnkZ%2FurhGy1fzdUw%2FqOvYRX7NGuySkSfSXVkw47h2zN6PeX3Qr6Jz26aAzXa3brkctO%2Be15%2BPj5YTB8evo9e9AmYG79G%2BjZo6yFvPYqastVOmI36NPcjB20mRSzI48De1iDlpjXLkIXU7QBhppVRp3DG7NXEW3kDG0isG9tIn0O0rfv74fPD%2F0%2FhoMv%2B8GW1ZbgqDSOI9iD4w18tcdcJG%2BGX98PSmIptQRLtXEslVxTTsXAteb0wNXacyY9W26utlxLHkLMGrrk7OB%2FaIVHHMvs%2F3KrjLPWOBH0Yk8pFPjGdXZ9l%2FjgEBfLPelfTQl6d8G2Y42g84jWNrbRkrS5%2Fmv2qRhtEiXcp44v7MmEPkvfcuwZbXDgNH7%2BddAcnpcPrQAINUNP9q%2BmkoFQ5iAIdK0qtyzbwT6MqGl%2BT03rF2tEu9gUpuRl8T4Xd4mWMKU8QRMDYUxECN19MPAIErGiV62m6T0taXI1PatpEg%2BnqjRNKrS541AUkU0FU%2B%2FDd5jv7TUua0APincON5qWOw4SDaJiypT7w0KdePjDRFoPfqpQ%2FD5ZC9gW7EDj2PFyLsdjdwOn1sbBD6MfNPnSEhC1xkHk%2BSrHg%2Bj1jm0Bz2wcPENs74ew5XzaLEbQfZj6erhuCZZhDNEclmZ1WN5ZS%2BI9tgZLtWksGZkEYXlHHvmbRQKDluqm0Tie%2BUl6IXi2Sz81qXE8s8H9qVHj%2B%2BUUtQU%2FpXH8slnzk6N%2BNH5pC3564%2FiJzdp8gLuWQKc3nrCRxSZsBpbj0AKUFiXd9MYTN9ximeMx9Echn798fxyWhfA8MuUCsJWU1MAFqxOJZ8S5IxeVgVti8ClvUGiEMEaLjjcc6Nd4UTFNrPU8lFlG%2FjEM01BgtAqRZTVpcicabFxsZ7QWr7sgfe9m1V1YLv0KR8TyQMyorul9KlFWYEjllFWuCk9QAk84mcGnYBe5eI5maGk5w6g1RfzonHtEMfLQ%2BwEx3gWWz9pgtE8%2FS40rrdHGHcN9VA1SHthyZxDvk0FgsOh77gXRhY6F7VeYeA7xlXDZ8piT5Qu3Nv47uJxuf6fbXTXYu9nGDt3s2M6SvI13UddUDNbgX6krgDVEF3t77Op8RL3zHqFrE3FRHd9vLI%2BCmS%2FXplDe%2B9hCg8RfODsONJAwnZIKGvZzQGEarrggR8sU5BwGezWFO7VyI9fDqoI0UknSKJWRpjDXdyHNuZHGKEkaozLSiE0oBhHV3fD6ZvjZ45RmLShAy9GafvW29DrS58OxaZi5%2BBc%2FgCRNpzSlkH6A63xSHf9W1LVuWXio9JJM1bLTDUjIWGeXyMucXqLDYy2Pma00rTk6NCoMRcLw43vsSGEooiYCkeODkMKY9viIhMWThYEnqwI9k5AEiE3cxWYJ9Laf2J3Wm9CzsGIGO9Zc1oyXnnB3uLOR9GWExzI6u4R5GLKRUXSDk9UT4WBwQ2PWIRyU9KkpDZGPUtg30FsTy%2F0a7pTSXa4kJNEaGVz6iGxPg8JS52QiMJxAwG7h247gqhS64WMcD7jYioNvRNXpDGNvHt%2BtQf93pdZ5Yzpz0wNMOWGDVlGqnq%2FUnGl21WdyT%2BveE517Vw47%2B5z%2BnZtKPDGXvC93WJhjBNXYj2vXtXaxE1bULqzzzUtItB1zEI0Unfw7CrUpYoPUQu9h1FbvIT3JXOqZzXoPDdsZqXPAmAbZSduL0PpISetTZHyqsDNK2chBaSpy4D62dmFA7QxQz4oB5oUBtTPgvIatpWNiyAsFTqOAfl4U4CwGcaFAxRQwzosC2YT%2FhQJVU8A8LwqACwXqpgALCc%2BFAsqFArVTQHjq6TQKXLIC9VOgsfFEPgUuaYH6KQDOiwL6hQK1U%2BC8coNStobkQoGqKXBeyUHefMwUJUQsWJdepC615lzhGnXpRe3qXaROlVJzg0ovUqeeDttkA%2B%2F%2FcqXPH77O9H%2BW8z8XOzi9atR2H6S2OePHeiXjxymlOnJAuZdVZf6Isn6i5h41gCyB1Bijmlo1veh8E6ToV8WAs9h1UPjDzOPThpl%2Ftmk2iprEEWRLUHlrnVZWxMK6ClGl76M7aE2gO0DoxYbkaqd9dUrMPwzLBzjr2bI0rmiMuf1Mo2Hir97PgLL9TLNzXsUuudLb7plikubaT1sIJIHkermSLHf1rDKrclfNanPUKh7MhoP%2BeOFhr6RCi5pXcIKmcmYN7NWVMwnu2ETmumrHkwZbPrZ4PFEvnqgkj5FCP4gVJ2DPH8QtQ4Zq6tOBclx9%2BsGFqil%2FU6ujTlXssqZR8bvUvhmI6bW%2FsmsPsSr4ekreG801Css0ptzL8ymDB2XNEjivwiTurPsUUeqdl5qahQodOMYuPa%2FroJk9fp5ZGK5j25SZK%2BhawUNtKahHeaSY3eJKMkT5oSrogmAirPdJLlrFckGJaaz0iqxhEDGRlRtiNlqPIM4uiPVShQSYTNPjJoELgdyoATi7Zct%2BJQOgml3TzNd%2FIzu7tW79%2F2mmwonW2UpUke%2FL66lJZyCNpqCYQeH%2FjKiYIV9iFY80nDgdviimyPsLqIIDi0w4HaW91MjcZGwLh8j5%2FkZqcqNW2URbshv95WGfS9GfdwbD%2FwE%3D"></iframe>
</div>

> Index = TypeIndex ^ OBJECT_HEADER adresinin LSB 2. baytı ^ nt!ObHeaderCookie

Örnek olması açısından bir objenin tipini yeni versiyona göre bulalım.

```cpp
lkd> !process 0 0 notepad.exe
PROCESS ffff948ed18e0340
    SessionId: 8  Cid: 5898    Peb: 1002a8000  ParentCid: 5130
    DirBase: 15cdbd002  ObjectTable: ffffe284a08f7d80  HandleCount: 236.
    Image: notepad.exe

lkd> !object ffff948ed18e0340
Object: ffff948ed18e0340  Type: (ffff948eb86d17a0) Process
    ObjectHeader: ffff948ed18e0310 (new version)
    HandleCount: 6  PointerCount: 195193
lkd> dt nt!_OBJECT_HEADER ffff948ed18e0310 -y TypeIndex
   +0x018 TypeIndex : 0x80 ''
lkd> db nt!ObHeaderCookie l1
fffff800`0aafc72c  84                                               .
lkd> ? 0x80 ^ 0x03 ^ 0x84
Evaluate expression: 7 = 00000000`00000007
```

Çıkan sonuçta gerçek index numarası olan 7 çıktısını almış alıyoruz. Bu objenin bir process objesi olduğunu yukarıdaki çıktıdan anlayabiliyoruz fakat hesaplarımızı sağlama almak almak için aşağıdaki gibi bir uygulama yapılabilir:

```cpp
lkd> dt nt!_OBJECT_TYPE poi(nt!ObTypeIndexTable + ( 7 * @$ptrsize ))
   +0x000 TypeList         : _LIST_ENTRY [ 0xffff948e`b86d17a0 - 0xffff948e`b86d17a0 ]
   +0x010 Name             : _UNICODE_STRING "Process"
   +0x020 DefaultObject    : (null)
   +0x028 Index            : 0x7 ''
   +0x02c TotalNumberOfObjects : 0x1ca
   +0x030 TotalNumberOfHandles : 0xed6
   +0x034 HighWaterNumberOfObjects : 0x1e0
   +0x038 HighWaterNumberOfHandles : 0x121e
   +0x040 TypeInfo         : _OBJECT_TYPE_INITIALIZER
   +0x0b8 TypeLock         : _EX_PUSH_LOCK
   +0x0c0 Key              : 0x636f7250
   +0x0c8 CallbackList     : _LIST_ENTRY [ 0xffffe284`97a91a00 - 0xffffe284`97a91a00 ]
```

Tüm bunları yapan fonksiyonu disassemble ederek incelemek istersek:

```cpp
lkd> uf nt!ObGetObjectType
nt!ObGetObjectType:
fffff800`0a43d370 488d41d0        lea     rax,[rcx-30h]
fffff800`0a43d374 0fb649e8        movzx   ecx,byte ptr [rcx-18h]
fffff800`0a43d378 48c1e808        shr     rax,8
fffff800`0a43d37c 0fb6c0          movzx   eax,al
fffff800`0a43d37f 4833c1          xor     rax,rcx
fffff800`0a43d382 0fb60da3f36b00  movzx   ecx,byte ptr [nt!ObHeaderCookie (fffff800`0aafc72c)]
fffff800`0a43d389 4833c1          xor     rax,rcx
fffff800`0a43d38c 488d0dedfa6b00  lea     rcx,[nt!ObTypeIndexTable (fffff800`0aafce80)]
fffff800`0a43d393 488b04c1        mov     rax,qword ptr [rcx+rax*8]
fffff800`0a43d397 c3              ret
```

Bu konu hakkında detaylı bir şekilde breakpoint kullanılarak debug yapılmayacaktır fakat verilen komutlara bakılırsa; 

1. İlk XOR komutuna baktığınızda RAX ile RCX, yani TypeIndex ile OBJECT_HEADER adresinin LSB 2. baytı XOR işlemine tutuluyor.

2. Daha sonra ObHeaderCookie değeri ECX register'ına atanarak önceki XOR çıktısı ile tekrar XOR işlemine tutuluyor.

3. Son adımda da ObTypeIndexTable'ın hafıza adresi RCX register'ına atanarak XOR ile elde edilen çıktı adresi @$ptrsize ile çarpılır ve çıkan sonuç alınan bellek adresine eklenir yani rax,qword ptr [rcx+rax*8]. Böylece objeye uygun OBJECT_TYPE yapısı elde edilmiş olunur.

## ~~Object Tracing~~

## ~~Object Lock~~

## Object Oluşma Bilgisi (CreateInfo)

ObjectCreateInfo bölümü, objenin yaratılma anında sağlanan bilgileri tutar. Bu bilgiler arasında, objenin hangi process tarafından yaratıldığı, hangi erişim haklarının verildiği, hangi dosya yolunu işaret ettiği gibi bilgiler yer alabilir.

```cpp
lkd> dt nt!_OBJECT_HEADER ffff948ecce15050 -y Obj*
   +0x020 ObjectCreateInfo : 0xffff948e`c4cb1d40 _OBJECT_CREATE_INFORMATION
lkd> dx ((nt!_OBJECT_CREATE_INFORMATION*)0xffff948e`c4cb1d40)
((nt!_OBJECT_CREATE_INFORMATION*)0xffff948e`c4cb1d40)                 : 0xffff948ec4cb1d40 [Type: _OBJECT_CREATE_INFORMATION *]
    [+0x000] Attributes       : 0x2141be0 [Type: unsigned long]
    [+0x008] RootDirectory    : 0x2991864 [Type: void *]
    [+0x010] ProbeMode        : 0 [Type: char]
    [+0x014] PagedPoolCharge  : 0x0 [Type: unsigned long]
    [+0x018] NonPagedPoolCharge : 0x0 [Type: unsigned long]
    [+0x01c] SecurityDescriptorCharge : 0x0 [Type: unsigned long]
    [+0x020] SecurityDescriptor : 0x0 [Type: void *]
    [+0x028] SecurityQos      : 0x0 [Type: _SECURITY_QUALITY_OF_SERVICE *]
    [+0x030] SecurityQualityOfService [Type: _SECURITY_QUALITY_OF_SERVICE]
lkd> dx ((nt!_OBJECT_CREATE_INFORMATION*)0xffff948e`c4cb1d40)->SecurityQualityOfService
((nt!_OBJECT_CREATE_INFORMATION*)0xffff948e`c4cb1d40)->SecurityQualityOfService                 [Type: _SECURITY_QUALITY_OF_SERVICE]
    [+0x000] Length           : 0x0 [Type: unsigned long]
    [+0x004] ImpersonationLevel : SecurityAnonymous (0) [Type: _SECURITY_IMPERSONATION_LEVEL]
    [+0x008] ContextTrackingMode : 0x0 [Type: unsigned char]
    [+0x009] EffectiveOnly    : 0x0 [Type: unsigned char]
```

# Kaynakça ve Referanslar

1. Windows Internals Seventh Edition Part 2 (2022), Microsoft
2. [A Light on Windows 10's “OBJECT_HEADER->TypeIndex”](https://medium.com/@ashabdalhalim/a-light-on-windows-10s-object-header-typeindex-value-e8f907e7073a)
3. [CodeMachine - Article - Windows Object Headers](https://codemachine.com/articles/object_headers.html)
4. [【旧文章搬运】Win7可变对象头结构之InfoMask解析 - 黑月教主 - 博客园](https://www.cnblogs.com/achillis/p/10183526.html)
5. [WINNT内核文档](https://systemroot.gitee.io/pages/apiexplorer)
6. [Vergilius Project \| Kernels](https://www.vergiliusproject.com/kernels)
