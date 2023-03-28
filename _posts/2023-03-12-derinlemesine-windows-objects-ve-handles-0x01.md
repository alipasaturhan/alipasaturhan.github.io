---
layout: post
title: Derinlemesine Windows Objects ve Handles (0x01)
date: 2023-03-12 00:18 +0300
categories: [Reverse Engineering, Windows]
tags: [reversing, windows, internal, object, handle]
---

![banner](/assets/img/2023-03-12-derinlemesine-windows-objects-ve-handles-0x01/banner.png)

# Objects ve Handles Bağlantısı

Windows nesneler kümesinden oluşan bir işletim sistemidir. Object, bir sistem kaynağını (resource) temsil eden bir data structure’dır. Uygulamalar, object data’larına veya bir object’in temsil ettiği sistem kaynağına doğrudan erişemez. Uygulamaların sistem kaynağını incelemek veya değiştirmek için kullanabileceği bir object handle edinmesi gerekir. Yani Bir process, object’i kullanmadan önce o object’in handle’ına sahip olmalıdır. Çünkü handle’lar (yalnızca) hedef object’lerin türünü/bellekteki adresini belirlemeye yardımcı olur.

Her handle'ın sistem tarafından tutulan bir tabloda bir girişi bulunur ve bu giriş handle'ın ilişkili olduğu objenin adresini, objenin türünü, handle'ın erişim haklarını ve diğer özelliklerini belirler. Bu tabloya "Object Table" denir ve Windows işletim sistemi tarafından yönetilir. Object Table, Windows işletim sistemi tarafından yönetilen bir veri yapısıdır ve normal kullanıcılar tarafından direk olarak görüntülenemez. Ancak, Windows API'leri veya benzer araçlar kullanarak, bu veri yapısındaki bilgilerin bir kısmına erişmek mümkün olabilir. Yazının başlarında fazla internal'e inmeden aşağıdaki gibi Object Table'ın structure'ı görüntülenebilir:

```cpp
lkd> dt nt!_HANDLE_TABLE
   +0x000 NextHandleNeedingPool : Uint4B
   +0x004 ExtraInfoPages   : Int4B
   +0x008 TableCode        : Uint8B
   +0x010 QuotaProcess     : Ptr64 _EPROCESS
   +0x018 HandleTableList  : _LIST_ENTRY
   +0x028 UniqueProcessId  : Uint4B
   +0x02c Flags            : Uint4B
   +0x02c StrictFIFO       : Pos 0, 1 Bit
   +0x02c EnableHandleExceptions : Pos 1, 1 Bit
   +0x02c Rundown          : Pos 2, 1 Bit
   +0x02c Duplicated       : Pos 3, 1 Bit
   +0x02c RaiseUMExceptionOnInvalidHandleClose : Pos 4, 1 Bit
   +0x030 HandleContentionEvent : _EX_PUSH_LOCK
   +0x038 HandleTableLock  : _EX_PUSH_LOCK
   +0x040 FreeLists        : [1] _HANDLE_TABLE_FREE_LIST
   +0x040 ActualEntry      : [32] UChar
   +0x060 DebugInfo        : Ptr64 _HANDLE_TRACE_DEBUG_INFO
lkd> ?? sizeof(nt!_HANDLE_TABLE)
unsigned int64 0x80
```

> Bu kaynaklar; sistem veya uygulama tarafından kullanılan bellek alanı, disk alanı, network bağlantıları vb. temsil eder. Örneğin, bir file objesi bellekteki bir dosya sistemine ait bir dosyayı temsil eder ve bu objeyi kullanan bir uygulamanın dosyaya erişebilmesini ve içeriğini okuyabilmesini veya değiştirebilmesini sağlar. 

## Neden Objects/Handles?

Sistem, iki ana nedenden dolayı sistem kaynaklarına erişimi düzenlemek için object’leri ve handle’ları kullanır:

1. Object’lerin kullanımı, Microsoft'un sistemi kolayca güncelleyebilmesini sağlar. Microsoft, Windows’un sonraki sürümlerini yayınladığında, güncellenen object’i çok az veya hiç ek çalışma olmadan yayınlamak.

2. Windows güvenliğinden yararlanmanızı sağlar. Her object’in kendi Access Control Lists’i (ACL) vardır. Bir uygulama object için her handle oluşturduğunda, sistem (Object Manager) object’in ACL'sini inceler. Ayrıca bir process, object üzerinde gerçekleştirebileceği eylemleri görmek için bu listeye bakar (process'lerin de object olduğunu unutmayalım).

Özet olarak objeler ve handle'lar, Windows işletim sistemi tarafından saklama, paylaşma, iletişim ve erişme işlemlerini kolaylaştırmak için kullanılır.

## Object Manager

Her obje bir header, bir body ve optional footer içerir. Tüm object’ler aynı struct’a sahip olduğundan, Windows'ta tüm objeleri koruyan tek bir Object Manager vardır. Object Manager sadece nesnenin başlığını yönetir ve body kısmı, nesne türüne göre bir Executive Sub-system tarafından yönetilir. Örneğin, bir file objesinin body kısmı, I/O Yöneticisi Executive Sub-system tarafından yönetilir. 

Aşağıda örnek olması açısından objenin header structure'ı görüntülenmiştir

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

Windows kernel’ı, kullanıcı modu process’leri tarafından kullanılmak üzere çeşitli nesne türleri tasarlanmıştır. Bu objelerin data structur’ları kernel space’te saklanır. Object Manager (internal adı “Ob”) windows kaynaklarını yöneten windows subsystem’idir. Kaynakları temsil eden tüm objelerin bir Object Type özelliği ve kaynakla ilgili diğer meta data’ları vardır. Tüm objeleri görmek için sysinternals paketindeki WinObj.exe aracını inceleyin.

> Object Manager direk olarak görülemez. Object Manager, Windows işletim sistemi tarafından yürütülen bir sistem yapı parçasıdır ve kullanıcılar tarafından doğrudan erişilemez.

## Bellek Yönetimi ve Performansı

Handle’lar ve object’ler sadece bellek parçalarıdır ve belleği tüketirler. Bu nedenle, sistem performansını korumak için handle’lar kapatılmalı ve artık ihtiyaç duyulmayan object’ler silinmelidir. Bu yapılmazsa, disk belleği dosyasının aşırı kullanımı nedeniyle uygulamanız sistem performansına zarar verebilir. 

Bir process sona erdiğinde, sistem otomatik olarak handle’ları kapatır ve process tarafından oluşturulan objeleri bellekten siler. Fakat bir thread sona erdiğinde, sistem genellikle handle’ları kapatmaz veya object’leri silmez. Bunun haricinde, eğer belirtilen object'tin header'ındaki flag'lerinde `OB_FLAG_PERMANENT_OBJECT` belirtilmişse, handle sayısı 0'a düştüğünde bile obje yok edilemez. Bir objenin flag bölümünde hangi flag'in seçildiğini bulabilmek için:

```cpp
lkd> dt nt!_OBJECT_HEADER -a Flags ffffa78f3b54d050
   +0x01b Flags : 0x2 ''
```

0x2'nin anlamı ise KernelObject, internal adı OB_FLAG_KERNEL_OBJECT olarak bilinen bir bayrak set edilmiştir. Kısaca bu objenin yalnızca kernel handle'larına sahip olabileceğini belirler. İleri kısımlarda bu konu hakkında daha fazla bilgi verilecektir.

<u>Handle'lar dinamik olarak oluşturulabilir ve yok edilebilir, ancak bir objenin yok edilmesi handle'ın otomatik olarak yok edilmesine neden olmaz. Uygulamalar, handle'ları kapatmak zorundadır, çünkü process terminate edilene kadar destroy edilen objenin handle'ları hala bellekte map edilmiş durumundadır</u>.

## Handle Limitations

Bazı objeler aynı anda yalnızca bir handle’ı destekler. Diğer objeler, tek bir obje için birden çok handle’ı destekler. İşletim sistemi, objenin son handle’ı kapatıldıktan sonra objeyi bellekten otomatik olarak kaldırır. Neden kaldırır? Hatırlayalım; “Eğer bir objenin referans sayısı 0 olduğunda, Object Manager objeyi bellekten tamamen kaldırabilir, çünkü artık kullanılmayan bir objedir.”

Object structure'ından ileri kısımlarda bahsedilmiştir fakat şimdiden bu header'daki referans counter'dan bahsetmekte fayda var. Object header'da bir objedeki tüm referans sayısını tutan bölüme PointerCount denir. Bu bölüm aracılığıyla hangi nesnelerin kullanımda olduğunun kaydını tutar (buradaki referans: handle sayısıyla ilişkilidir).

> Her objenin bir referans sayıcı vardır ve her objeye erişen işlem, objenin referans sayısını bir arttırır. Eşzamanlı olarak, objeyi kullanmaktan vazgeçildiğinde, objenin referans sayısı bir azaltılır. Bu sistem, Windows işletim sistemi tarafından yönetilen objelerin bellek yönetimini düzenlemek için kullanılır. Bir objenin referans sayısı 0 olduğunda, Object Manager objeyi bellekten tamamen kaldırabilir, çünkü artık kullanılmayan bir objedir. Ayrıca şu bilgiyi de unutmayalım: “Bir process sona erdiğinde, sistem otomatik olarak handle’ları kapatır ve process tarafından oluşturulan objeleri bellekten siler.” Bu işlem güvenli bellek yönetimi içindir.

Aşağıda bir objenin yapısında referans sayısını tutan bölümü görebilirsiniz:

```cpp
lkd> dt nt!_OBJECT_HEADER -a PointerCount
   +0x000 PointerCount : Int8B
```

Bir objenin header adresini vermek gerekirse:

```cpp
lkd> dt nt!_OBJECT_HEADER ffffa78f39e40050 -a PointerCount
   +0x000 PointerCount : 0n261879
```

Sistemdeki toplam açık handle sayısı, kullanılabilir bellek miktarıyla sınırlıdır. Bazı obje türleri, session veya process başına sınırlı sayıda handle’ı destekler. Bir process aynı anda en fazla 16.000.000 handle’a sahip olabilir. Bu sınırlama, işletim sistemindeki bellek ve kaynakların yönetimi için gereklidir ve farklı uygulamaların aynı anda fazla bellek ve kaynak kullanmasını önlemek amacıyla uygulanır. Yani her bir uygulama için, bellekteki nesneler için belirli bir sayıda handle verilir ve bu handle’lar, uygulamanın bellekteki objelere erişmesini sağlar. Eğer uygulama, verilen handle sayısını aşarsa, işletim sistemi tarafından "handle is out of range" (işaretçi aralık dışında) hatası verilir ve uygulamanın bellekteki objelere erişimi engellenir. Bu nedenle, uygulama yazarları, handle sınırlamalarını göz önünde bulundurmalı ve bellek yönetimine uygun olarak işletmeleri gereken handle'ları tasarlamalıdır.

```cpp
#include <windows.h>
#include <iostream>
#include <psapi.h>

int main() {
    DWORD processId = 14388; // replace with the actual process ID
    HANDLE hProcess = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, FALSE, processId);
    if (hProcess == NULL) {
        std::cout << "Failed to open process. Error code: " << GetLastError() << std::endl;
        return 1;
    }

    DWORD dwHandleCount = 0;
    BOOL result = GetProcessHandleCount(hProcess, &dwHandleCount);
    if (!result) {
        std::cout << "Failed to get handle count. Error code: " << GetLastError() << std::endl;
        CloseHandle(hProcess);
        return 1;
    }

    PROCESS_MEMORY_COUNTERS memCounters;
    result = GetProcessMemoryInfo(hProcess, &memCounters, sizeof(memCounters));
    if (!result) {
    std::cout << "Failed to get process memory info. Error code: " << GetLastError() << std::endl;
        CloseHandle(hProcess);
        return 1;
    }

    CloseHandle(hProcess);

    std::cout << "Current handle count: " << dwHandleCount << std::endl;
    std::cout << "Memory used by handles: " << memCounters.WorkingSetSize << " Kb" << std::endl;

    return 0;
}
```

Veya örnek olması açısından seçtiğimiz bir process object'in handle sayısını debugger ile görüntülemek istediğimizde HandleCount bölümünün 0n7 olduğunu görebiliriz:

```cpp
lkd> dt nt!_OBJECT_HEADER ffffa78f39e40050 -a PointerCount HandleCount
   +0x000 PointerCount : 0n261513
   +0x008 HandleCount  : 0n7
```

Windows sürümlerinde handle limiti varsayılan olarak belirli bir değere ayarlanır ve bu varsayıılan limit değeri, windows, sürüm ve sistem yapılandırmasına göre değişebilir.

# Handle Yönetimi

Windows, objeler için aşağıdaki görevleri gerçekleştiren işlevler sağlar:

- Creating an object [*CreateEvent(), CreateProcess(), CreateThread(), ...*]

- Get/Set an object handle [*GetHandleInformation() / SetHandleInformation()*]

- Close the object handle [*CloseHandle()*]

- Open an object and get handle [*OpenEvent(), OpenProcess(), OpenThread(), ...*]

- Destroy the object [*CloseHandle() (`OB_FLAG_PERMANENT_OBJECT` set edildiyse process terminate edilmelidir)*]

Bu fonksiyonları sağlayan handleapi.h header'ı görebilirsiniz:

```cpp
/**
 * This file is part of the mingw-w64 runtime package.
 * No warranty is given; refer to the file DISCLAIMER within this package.
 */
#ifndef _APISETHANDLE_
#define _APISETHANDLE_

#include <apiset.h>
#include <apisetcconv.h>
#include <minwindef.h>

#ifdef __cplusplus
extern "C" {
#endif

#define INVALID_HANDLE_VALUE ((HANDLE) (LONG_PTR)-1)

#if WINAPI_FAMILY_PARTITION (WINAPI_PARTITION_APP)
  WINBASEAPI WINBOOL WINAPI CloseHandle (HANDLE hObject);
  WINBASEAPI WINBOOL WINAPI DuplicateHandle (HANDLE hSourceProcessHandle, HANDLE hSourceHandle, HANDLE hTargetProcessHandle, LPHANDLE lpTargetHandle, DWORD dwDesiredAccess, WINBOOL bInheritHandle, DWORD dwOptions);
#endif

#if WINAPI_FAMILY_PARTITION (WINAPI_PARTITION_DESKTOP)
  WINBASEAPI WINBOOL WINAPI GetHandleInformation (HANDLE hObject, LPDWORD lpdwFlags);
  WINBASEAPI WINBOOL WINAPI SetHandleInformation (HANDLE hObject, DWORD dwMask, DWORD dwFlags);
#endif

#ifdef __cplusplus
}
#endif
#endif
```

## Source Code Annotation Language (SAL)

Öncelikle handle'ları yöneteceğimiz fonksiyonları anlayabilmek için parametrelerine bakmamız gerekli, parametrelerini anlayabilmemiz için; SAL yani Source Code Annotation Language yani Kaynak Kodu Ek Açıklama Dili’ni öğrenmeliyiz. Aşağıdaki tablo yardımıyla çok kısa ve hızlı bir bilgi verilmiştir:

| Parametre  | Açıklaması                                                                           |
| ---------- | ------------------------------------------------------------------------------------ |
| _ In _     | Çağrılan fonksiyona veri iletilir ve readonly modundadır                             |
| _ Inout _  | Çağrılan fonksiyona kullanılabilir veriler aktarılır ve read and write modundadır    |
| _ Out _    | Çağrılan fonksiyon verileri bu boşluğa yazar. Gelen veriye sadece yazma işlemi yapar |
| _ Outptr _ | Çağrılan fonksiyon tarafından return edilen değer bir pointer’dır                    |

> Ek olarak; _ Opt _ ise tabloda gösterilen notasyonların sonuna gelir. Çağrılan fonksiyon isteğe bağlı veri girişi aktarılacağını belirtir. Örneğin: _ Inopt _ veya _ Outopt _ gösterilebilir.

## DuplicateHandle()

```cpp
BOOL DuplicateHandle(
  [in]  HANDLE   hSourceProcessHandle,
  [in]  HANDLE   hSourceHandle,
  [in]  HANDLE   hTargetProcessHandle,
  [out] LPHANDLE lpTargetHandle,
  [in]  DWORD    dwDesiredAccess,
  [in]  BOOL     bInheritHandle,
  [in]  DWORD    dwOptions
);
```

Objeleri temsil eden handle’ların duplicate edilmesini sağlar. Bu fonksiyon objeden objeye farklılık gösterebilir. Çünkü bildiğiniz üzere user objeleri yalnızca tek bir handle alabiliyorken, kernel objeleri birden fazla handle alabiliyordu. Bu fonksiyonun parametreleri hızlıca açıklanmak istenirse, sırayla:

- Duplicat edilecek handle’ın hangi process’te olduğu.

- Belirtilen process’teki hangi handle’ın duplicate edileceği.

- Seçilen process’teki seçilen handle’ın hangi process için duplicate edeceği.

- Son olarak duplicate edilmiş handle’ı yerleştirmek için seçilen process’teki bir handle objesi.

Bu şekilde anlatılınca biraz karmaşık gelmiş olabilir. Aşağıdaki görselde bu fonksiyonun nasıl çalıştığı ifade edilmeye çalışılmıştır:

<div>
<iframe width="100%" height="300" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Untitled%20Diagram%20(2).drawio#R7VpbU9s4FP41mdk%2BwNjyJcljLhQeujvM0Nlt92VH2MLWoFiu7JCkv34lW7It2alD6kBoIQNIR9LR5Xzf0dGBkbNYba8ZTOM%2FaYjICFjhduQsRwDYlmfxX0KyKyUesEtBxHAoO9WCO%2FwdqZFSusYhyrSOOaUkx6kuDGiSoCDXZJAxutG7PVCiz5rCCLUEdwEkbek%2FOMzjUjpR2xLyG4SjWM1sW7LlHgaPEaPrRM43Ao5VfJXNK6h0SUEWw5BuGiLnauQsGKV5WVptF4iIs1XHVo77uKe1WjdDSX7IgL%2FTWbgAs3%2Bv0fTJSb2bbzPoXkgtT5CskdpGsdh8pw6o2CISSuyRM9%2FEOEd3KQxE64ZDgsvifEVk8wNN8gUllBVjnY%2BW%2BHA5gfeIzKsTa3SRZ%2BbMs5zRR9QcXHwJpZiQrhHtE1DbQSxH24ZInsg1oiuUsx3vsq3MWQ6R6LWVtTY1FoAyadzAAXClEEr8RZXu2ga8IM3wDJOAfpOgJJwJ6PNaQhMunIcwiysbNexRG08cGNri%2FEuj%2FFWUL4Enq8tto225U5WE7%2BqL1FdUmsNEvR5X1NTAk2Bhr80zumYB6od6DlmE8v5%2BKNT8RhtBDYR4HQBRMoYIzPGT7m26QCNnuKWY76wCqGvpAHUcA3flvuWoJv1NRZ6uCEwMReXBtBRxoMFdo1sqOmT7Fwx8g1ETzS3xQqmxJkh1psdzxmlx5pbRAGV8zTMu%2F4%2F%2F4Ip9wm0zv2e8FInSDUxCQamiSzxr0Yz7kFynk45Lyb2me5IiSHCU8GrAUYq4fC48EuaXzkw2rHBYTN3pT3XS%2FgSNhnCQpjkdv%2BUg7S4HaeJ0MP%2Fo%2F%2BZXlued3ZU17qHf%2FAD6zd%2Fp12Fs3zo3%2Bk3ew5NjwpMzCSdazsOMAg4NJ8Z9igYKJ%2FzJK4QTdtezqHRfsa3c13KdEu5TclT6sT9UDz5j3anyeoQGj9%2FWNEcaVXwh4s0rflg4KV2h1fh2rXRbaLHuKQuFKxM9JKkKoIUhTiI5UPZVSs01pJ1TE5ygC4U0oaZ4EnmdmuK7Eh2lc5f%2Bu952erIZX2SqzyViX25zJC2n7J%2BLy5oIGnohFUo7NQlHeZEViRuhB7h7cPahudqWataxp8Y%2BB7v7M36ZF5RYenXtM%2BUX%2B%2FICWHuufMpv5AdS3FgxDw1QcmZhQPXqU15QeV8t5vM6Yj77VHGA3ZWneBYge%2BEozvwQRNpgDyJvZn8tP12J8HLRQF2p9hlg5KeB02xfuNjAKMzSMkX5gLcCOcVCZcoTuG%2FwrWFPbA131ROiGX4qiMV6mvRUuGu%2F9c8Yd8t33B31xp16Ou6s18ed28KdGQF%2B6DJiFsNUFGO0hREVF0uKGOZrEtGckt4qEeg3dmFjaVxbt7UthmeyfLI3zin9zdjX77lph91Bh91Pltqwvf73rjJxsGZkN2f8WEWCuc%2BOepKqsMgtzXCOBUiWBD0IHSpx8clorhIYKtXByrPYn%2BnYAxPr0uvAyUJ8foiTCg1HvIWfgQevHw9dcKiylsPjoZ1%2BvAxoiAaMYIfOXjWt%2FsrJZEs3J%2FDcljn9DnOeLJllH5LNCiN0J6uI3NPNVS2YFwLhxynD3%2FnZQiKEdf4rIDDLcKADgfuJp8pCRgosUa59UiAGsrxRN625L%2Fc1fm7uS1H2NHzv%2FZObDOnOJEfm%2BGbwMdFVHJojA2buyjUUDZQjc4Axz4vkyKa%2FJHXOiznnyggHXHpHcmJqvCinzotwAliuXPJJWQG6MsdvkBXW275QxmdFH88CRtgzPo48nmcosgxFA5HHcw3y2FbPun7Y%2F0RU60pAvn2qvTGm2ecVu7m%2BqyNxYlwvB1PN16lmT4wLbyiqjQ3qjH%2BKOrxa%2F0dp2b3%2Bt13n6n8%3D"></iframe>
</div>

Görselde de ifade edildiği gibi DuplicateHandle fonksiyonu _A process’inde execute edilerek _C adındaki handle’ını _B process’ine çoğaltıyor. _B process’inde önceden var olan veya yeni boş bir handle oluşturup o handle hedef olarak veriliyor. Böylece _C handle’ı _D handle’ı adı altında duplicate ediliyor. Ayrıca bir handle duplicate edilecekse, başka bir process’teki handle hedef olarak verilmek zorunda değil. DuplicateHandle fonksiyonuna hem source process hem de target process, current process’in handle’ı verilirse, tek bir process’te aynı objeye sahip iki farklı handle oluşturulabilir. 

Bu arada dikkat etmenizi istiyorum, DuplicateHandle fonksiyon parametresinde bir tek `lpTargetHandle` parametresinin tipi farklıdır. Diğer parametreler direkt olarak handle’ın kendisini alırken, bu parametre handle’ın bellekteki adresini alıyor.

Fonksiyonun diğer parametrelerine bakacak olursak: dwDesiredAccess, bInheritHandle ve dwOptions parametreleri vardır. Bunların mantığını hızlıca anlatmaya çalışırsak:

- `bInheritHandle`: Duplicate edilen handle’ın process’ler arası inherit edilebilirliğini belirtiriz.

- `dwOptions`: Duplicate edildikten sonra source handle’ın kapatılıp kapatılmayacağını belirtiriz. Bu parametreye iki tür değer girilebilir: “DUPLICATE_CLOSE_SOURCE” (0x1: source handle’ı kapatır) ve “DUPLICATE_SAME_ACCESS” (0x2: source handle’ı kapatmaz).
  
  Eğer 0x2 seçilirse dwDesiredAccess parametresi görmezden gelinecektir. Mantığını da siz anlayın :)

- `dwDesiredAccess`: Bu parametre handle’ın temsil ettiği objenin türüne bağlı olarak değişebilen bir parametredir. Yani bu parametreye bir çok değer girilebilir. Eğer dwDesiredAccess’i kullanacaksak, uğraştığınız handle’ın tuttuğu objeye hakim olmanız gerekir. Parametrenin adından da anlaşılacağı gibi, duplicate edilmiş handle’a diğer handle’dan (source handle) farklı erişim yetkileri vermemizi sağlar. Aşağıda objelere göre gelebilecek örnek değerler listelenmiştir:
  
  > File Objesi’nin Handle’ı İçin: GENERIC_READ, GENERIC_WRITE, GENERIC_EXECUTE, DELETE, FILE_READ_ATTRIBUTES, FILE_WRITE_ATTRIBUTES, FILE_READ_DATA, FILE_WRITE_DATA
  > 
  > Registry Key Objesi’nin Handle’ı İçin: KEY_READ, KEY_WRITE, KEY_ALL_ACCESS

Eğer hTargetProcessHandle parametresi NULL ise; lpTargetHandle, dwDesiredAccess ve bInheritHandle parametreleri görmezden gelinecektir.

## CloseHandle()

```cpp
BOOL CloseHandle(
  [in] HANDLE hObject
);
```

Handle’lar belleğin baş belası olduğunu biliyorsunuz, (aslında optimizasyon sistemleri sayesinde baş belası olmaktan çıktılar gibi) bu fonksiyon hObject parametresindeki belirtilen handle’ın bellek hücresini serbest bırakır ve handle’ı kapatır. Bu işlem yapıldıktan sonra artık belirtilen handle kullanılamaz. Aşağıda CloseHandle ile bir kernel objesinin handle'larını kapattığımızda geçen senaryo anlatılmaktadır:

![close_handle_diagram_01](/assets/img/2023-03-12-derinlemesine-windows-objects-ve-handles-0x01/close_handle_diagram_01.png)

Şimdi bu diyagrama göre; varolan bir event objesine ait iki farklı uygulamada handle'a sahip olduğumuzu düşünebiliriz. 

1. İlk uygulama CloseHandle fonksiyonunu call ederek handle için kapama isteği yollanıyor.

2. Sistem gelen çağrıyı işler ve belirtilen handle kapatılır.

3. İkinci uygulama da birinci uygulamanın yaptığını yapar ve sahip olduğu handle'ı için kapama isteği yollar.

4. Sistem, event objesine ait son handle'ı da kapatır.

5. Object manager, event objesinin 0 adet handle'ı kaldığı için objeyi bellekten siler.

## GetHandleInformation()

```cpp
BOOL GetHandleInformation(
  [in]  HANDLE  hObject,
  [out] LPDWORD lpdwFlags
);
```

Bu fonksiyon hedef handle için aşağıdaki HANDLE_INFORMATION yapısını kullanır. Bu yapı handle’ların pointer biçimindeki özelliklerini içerir. Bu özellikler, HANDLE_FLAG_INHERIT ve HANDLE_FLAG_PROTECT_FROM_CLOSE olarak adlandırılır. 

GetHandleInformation, bir handle ve bir HANDLE_INFORMATION yapısı alır. Fonksiyon, handle’ın özelliklerini HANDLE_INFORMATION yapısına yazar ve başarı durumunda TRUE değerini döndürür. Başarısızlık durumunda ise FALSE döndürür ve GetLastError kullanılarak hata kodu elde edilir.

> HANDLE_INFORMATION yapısı, bir handle hakkında bilgi taşıyan bir yapıdır. Bu yapı, bir handle’ın işaretçi biçimindeki özelliklerini tutar ve handle’ın içine dahil bir özellik değildir. Aşağıda HANDLE_INFORMATION yapısının C++ struct kodunu görebilirsiniz:
> 
> ```cpp
> typedef struct _HANDLE_INFORMATION {
>     DWORD dwFlags;
>     HANDLE handle;
> } HANDLE_INFORMATION, * PHANDLE_INFORMATION;
> ```

Aşağıda bu fonksiyon için örnek bir kod görebilirsiniz:

```cpp
#include <Windows.h>
#include <stdio.h>
#include <iostream>
int main() {
    HANDLE hFile = CreateFile(L"C:\\Users\\adel\\Desktop\\example.txt", GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile == INVALID_HANDLE_VALUE) {
        DWORD dwError = GetLastError();
        LPSTR messageBuffer = nullptr;
        size_t size = FormatMessageA(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
            NULL, dwError, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPSTR)& messageBuffer, 0, NULL);
        std::string message(messageBuffer, size);
        LocalFree(messageBuffer);
        std::cout << "CreateFile failed with error: " << message << std::endl;
    }
    else {
        DWORD dwFlags;
        if (!GetHandleInformation(hFile, &dwFlags)) {
            printf("Error: failed to get handle information\n");
            CloseHandle(hFile);
            return 1;
        }
        printf("Handle information for example.txt:\n");
        printf("    Handle flags: 0x%08x\n", dwFlags);
        if (dwFlags & HANDLE_FLAG_INHERIT) {
            printf("    Inheritable handle: yes\n");
        }
        else {
            printf("    Inheritable handle: no\n");
        }
        CloseHandle(hFile);
    }
    return 0;
}
```

HANDLE_INFORMATION yapısına gelen değerleri öğrenmek için SetHandleInformation’a bakınız.

## SetHandleInformation()

```cpp
BOOL SetHandleInformation(
  [in] HANDLE hObject,
  [in] DWORD  dwMask,
  [in] DWORD  dwFlags
);
```

SetHandleInformation fonksiyonu, bir handle’ın öznitelikleri değiştirmek için kullanılır. Bu fonksiyon, bir handle hakkında bilgi almak için kullanılan `GetHandleInformation` işlevi ile birlikte kullanılabilir.

- `hObject`: özniteliklerinin değiştirileceği handle.

- `dwMask`: değiştirilecek özniteliklerin belirtildiği bit maskesiyle belirtilir.

- `dwFlags`: ayarlanacak özniteliklerin değeriyle belirtilir.

dwMask parametresi, değiştirilecek öznitelikleri belirtir. Örneğin, HANDLE_FLAG_INHERIT özniteliği değiştirilecekse, dwMask parametresine HANDLE_FLAG_INHERIT değeri atanır.

dwFlags parametresi, ayarlanacak özniteliklerin değerini belirtir. Örneğin, bir saplama(handle) için HANDLE_FLAG_INHERIT özniteliği kapatılmak isteniyorsa, dwFlags parametresine 0 değeri atanır.

```cpp
// Handle'ı inheritable yapmak
SetHandleInformation(hFile, HANDLE_FLAG_INHERIT, HANDLE_FLAG_INHERIT);

// Handle'ı uninheritable yapmak
SetHandleInformation(hFile, HANDLE_FLAG_INHERIT, 0);

// Handle'ı kapatılamaz hale getirmek: 
SetHandleInformation(hFile, HANDLE_FLAG_PROTECT_FROM_CLOSE, HANDLE_FLAG_PROTECT_FROM_CLOSE);

// Handle'ı hem kapatılamaz hale getirip hem de inheritable yapmak:
SetHandleInformation(hFile, HANDLE_FLAG_INHERIT | HANDLE_FLAG_PROTECT_FROM_CLOSE, HANDLE_FLAG_INHERIT | HANDLE_FLAG_PROTECT_FROM_CLOSE);
```

Bu arada dwMask ve dwFlags’e gelen değerler HANDLE_INFORMATION yapısından geliyor.

## CompareObjectHandles()

```cpp
BOOL CompareObjectHandles(
  [in] HANDLE hFirstObjectHandle,
  [in] HANDLE hSecondObjectHandle
);
```

Bu fonksiyon, iki handle'ın aynı nesneyi temsil edip etmediğini belirlemek için kullanılır. İki handle aynı nesneyi temsil ediyorsa, fonksiyon TRUE döndürür. Aksi takdirde, fonksiyon FALSE döndürür.

İki handle'ın aynı nesneyi temsil etmesi, aynı işlem tarafından açılmış iki handle'ın birbirine eşit olması anlamına gelir.

## Handle Inheritance

Bir child process, parent process’inden handle’lar alabilir. Ama başka inherit alamaz. Bir handle’ın inherit edilebilmesi için 2 şey yapılmalıdır:

1. SECURITY_ATTRIBUTE yapısının bInheritHandle üyesi set edilmelidir.

2. CreateProcess cağrılırken bInheritHandles parametresi TRUE olarak ayarlanmalı 
   
   (ek olarak STD:I/O/ERR handle’ları alabilmek için STARTUPINFO yapısının dwFlags üyesi STATF_USESTDHANDLES içermelidir).

Bir child process’in devralıması gereken handle’ların bir listesini belirtmek için UpdateProcThreadAttribute fonksiyonunun PROC_THREAD_ATTRIBUTE_HANDLE_LIST flag ile kullanılmalıdır. Bir handle’ın inherit edilip edilmediğini anlayabilmek için SetHandleInformation fonksiyonu kullanılabilir.

> Process sadece bu objeleri inherit veya duplicate edebilir: Access Token, Communications device, Console input, Console screen buffer, Desktop, Directory, Event, File, File mapping, Job, Mailslot, Mutex, Pipe, Process, Registry key, Semaphore, Socket, Thread, Timer, Window station.

Aşağıda örnek c++ kodu ile handle inheritance görebilirsiniz:

```cpp
#include <windows.h>
#include <iostream>

int main()
{
    HANDLE fileHandle;
    SECURITY_ATTRIBUTES sa;
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
    HANDLE childStdOut;
    DWORD bytesWritten;
    char buffer[100];
    char* command = "cmd.exe /C echo hello world";

    // Dosya işlemi için güvenlik özniteliklerini belirleme
    sa.nLength = sizeof(SECURITY_ATTRIBUTES);
    sa.lpSecurityDescriptor = NULL;
    sa.bInheritHandle = TRUE;

    // Dosya işlemi için handle oluşturma
    fileHandle = CreateFile("output.txt", GENERIC_WRITE, 0, &sa, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
    if (fileHandle == INVALID_HANDLE_VALUE)
    {
        std::cout << "Failed to create file" << std::endl;
        return 1;
    }

    // Başlatılacak işlem için gerekli STARTUPINFO ve PROCESS_INFORMATION yapılarını belirleme
    ZeroMemory(&si, sizeof(STARTUPINFO));
    si.cb = sizeof(STARTUPINFO);
    si.dwFlags = STARTF_USESTDHANDLES;

    // Dosya işlemi handle'ının standart çıkış olarak kullanılacağını belirtme
    si.hStdOutput = fileHandle;
    si.hStdError = GetStdHandle(STD_ERROR_HANDLE);
    si.hStdInput = GetStdHandle(STD_INPUT_HANDLE);

    // Yeni işlem başlatma
    if (!CreateProcess(NULL, command, NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi))
    {
        std::cout << "Failed to create process" << std::endl;
        return 1;
    }

    // Başlatılan işlemin çıkışını dosyaya yazma
    childStdOut = pi.hProcess;
    while (ReadFile(childStdOut, buffer, sizeof(buffer), &bytesWritten, NULL) && bytesWritten != 0)
    {
        WriteFile(fileHandle, buffer, bytesWritten, &bytesWritten, NULL);
    }

    // Dosya işlemi ve işlem handle'larını kapatma
    CloseHandle(fileHandle);
    CloseHandle(childStdOut);
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);

    return 0;
}
```

Burada CreateFile API fonksiyonu kullanılarak "output.txt" adında bir dosya oluşturuluyor ve bu dosya işlemi, sa.bInheritHandle parametresini TRUE olarak belirterek miras alınacak şekilde oluşturuluyor. Daha sonra, STARTUPINFO yapıları belirtiliyor ve si.hStdOutput parametresi, dosya işlemi handle'ını standart çıkış olarak belirterek işleme miras olarak aktarılıyor. Son olarak, ReadFile ve WriteFile API fonksiyonları kullanılarak işlem çıktısı dosyaya yazılıyor ve bellekte boş yer kaplamamaları için handle'lar kapatılıyor.

## ~~Handle Duplication~~

Aşağıdaki örnek kod ise DuplicateHandle fonksiyonu ile bir event handle’ını çoğaltmayı sağlıyor:

```cpp
#include <windows.h>
#include <iostream>

int main() {
    HANDLE hEvent = CreateEvent(NULL, TRUE, FALSE, NULL);
    if (hEvent == NULL) {
        std::cerr << "CreateEvent failed, error code: " << GetLastError() << std::endl;
        return 1;
    }

    HANDLE hEventWaitOnly = NULL;
    if (!DuplicateHandle(GetCurrentProcess(), hEvent, GetCurrentProcess(), &hEventWaitOnly,
        SYNCHRONIZE, FALSE, 0)) {
        std::cerr << "DuplicateHandle failed, error code: " << GetLastError() << std::endl;
        CloseHandle(hEvent);
        return 1;
    }

    CloseHandle(hEvent);
    if (WaitForSingleObject(hEventWaitOnly, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForSingleObject failed, error code: " << GetLastError() << std::endl;
        CloseHandle(hEventWaitOnly);
        return 1;
    }

    CloseHandle(hEventWaitOnly);
    return 0;
}
```

# Object Kategorileri

Aslında objeler kategorilere ayrılmaz. Çünkü hepsinin yapısı aynı ve sistemde kaynakları yönetmek için kullanılır. Fakat sistem içinde objelerin bazılarının erişim yetkileri farklıdır. Bu yüzden objeleri üç çeşit kümeye ayırabiliriz:

- Executive Objects

- Kernel Objects

- User/GDI Objects

Temelde Executive ile Kernel nesne türü arasındaki fark, bir Kernel Object'in doğrudan işletim sistemi çekirdeği tarafından yönetilmesi olurken, bir Executive Object'in Object Manager tarafından yönetilmesidir.

## Executive Objects

- İşletim sistemi çekirdeği tarafından yönetilir fakat kernel objeleri gibi doğrudan kernel modu komutları almazlar

- Kullanıcı modunda oluşturulurlar  ve çekirdek üzerinde çalışırlar, yani kullanıcı modunda işlem yaparlar

- Uygulamaların kullanabileceği yüksek seviyeli nesnelerdir ve uygulamalar genellikle bu nesnsleri kullanarak kaynakları yönetirler

- Bellekte dinamik olarak oluşturulur ve özel alanlarda yönetilir

## Kernel Objects

- Windows tarafından uygulanan daha ilkel bir nesne kümesidir

- İşletim sistemi çekirdeği tarafından yönetilir ve doğrudan kernel modu komutları alırlar

- Kernel modunda oluşturulurlar ve çekirdek altında çalışırlar, yani kernel modunda işlem yaparlar

- Kullanıcı modunda görünmez, yalnızca executive içinde oluşturulur ve kullanılır. Bu özelliği sayesinde de senkronizasyon gibi yetenekler sağlar

- Böylece, birçok executive objesi bir veya daha fazla çekirdek nesnesini kapsar (enkapsüle edilir)

- Kernel tarafından yönetilen bellek havuzlarında saklanır. Bu bellek havuzları, işletim sistemi tarafından dinamik olarak yönetilir ve her havuzda farklı tiplerde nesneler saklanabilir.

## Executive/Kernel Objesinin Bağları

Bazı nesneler hem Executive Obje hem de Kernel Objesi olarak kullanılabilirler. Çünkü uygulamalar ve işletim sistemi, bu nesnsleri kullanarak bilgi paylaşabilir ve senkronize edebilir. Bu yüzden bir executive objesi için “kernel modunda oluşturulabilir” gibi bir ifade görürseniz, bunu bir kernel objesi olarak sayabilirsiniz.

![executive_kernel_object_preview_01](/assets/img/2023-03-12-derinlemesine-windows-objects-ve-handles-0x01/executive_kernel_object_preview_01.png)

Kernel objelerinin ve executive objelerinin direkt olarak erişebildikleri yerleri bir diyagram olarak anlatmak istediğimizde:

<div>
<iframe width="100%" height="300" src="https://viewer.diagrams.net/?tags=%7B%7D&highlight=0000ff&edit=_blank&layers=1&nav=1#R7VnbdqIwFP0aHjsLiKB9VMS203bskl4fU4nAiIbGeOvXT4BEburQVZCu1fICZ%2BdCOPvscwJIwJhtLggM3FtsI19SZXsjgb6kqoqsyewUItsY0TpaDDjEs3mnBLC8dyRGcnTp2WiR6Ugx9qkXZMExns%2FRmGYwSAheZ7tNsJ%2B9awAdVACsMfSL6JNnUzdGO%2BKxQvwSeY4r7qzIvOUVjqcOwcs5v5%2BkAjk64uYZFHNxYOFCG69TEDAlYBCMaXw12xjID30r3BaPGxxo3a2boDktM2Bq6Gfw7WL7cjWcBd3H3qrvBWd8lgXdCn9gQl3s4Dn0bzAOGKZIoPcXUbrl3MElxQxy6cznrWjj0Wd2Lf%2FSNG6%2BROa5MPthuMjC2KaMO0S8GaKICGxOyZZPJsyXdFsyVWSJuRaU4CkysI9J9BxgEB2sJaII2XytjIUgfArfm09jMyYlnGPN%2BQ%2BvY78guxA8ibuF7%2FCSjHmva3hz3R4ZVwPSejPOX%2B%2B674YlfEwhcRA91k%2FdRQVTG8LMLWTLBhLkQ%2BqtsguBPOydXb%2BEenbB2f9AJIBCJCSOCx3yqbjotLOB8Zm4yERFEiQfjotcJDRMfutE3B9b5Ar6SySSme6z5fYmmD1wmHyF%2F%2FS3ZZiwmIvAJDrSkO6E52tz9Me8ETO8EoHfDvumQNky46njpkLssUWzAoBSNI19vLRDnlyPIiuAkdfXrCZlY27i%2BX6KbXnPcTQqDjK9QoSiTQoqksVbVZlXQF4R1RavAOukviiiKrip2tKRa9J2%2B0fbdWhbLaltrcm8LvZZ9ZDfYMHPFfKDsXA68vf6%2F7zJxK5WmtgfLHP0fdO6pubSOtAbTuvqEWUrn1O2In0gpScKVvRcBv9fCo%2BsfEpoOq%2BDshv2JqUNKpX2ZXfUf%2BqOSgs5W0JKiNfiY3nIfQE9t3Jy1lrl5KzXJWe9vkL9reXcKinnU71%2BH1tkLa9glYu5YeUCPS%2Fdki9YVUj38A6%2FMvas4eC%2B1mTcMH96J%2F%2BCXHInVRt%2Fyr6N8tG91df3sprbrwK9qJKdR6ver%2B73crV7lkPfmYa936Zxb51OPbUzCXJMtpvOd0q15cp8No2H%2B6tH8wuQeeJSpjVI7eHvFT%2FM7qVxD9mHi5z4EXgCZpmZ%2FEeM2lI%2Fa4H5Dw%3D%3D"></iframe>
</div>

Yani, executive objelerinin donanımla doğrudan iletişim kurmadıklarını ancak kernel modundaki işlevler aracılığıyla donanımla etkileşim sağladıklarını söyleyebiliriz.

## User/GDI Objects

Windows işletim sistemi, kullanıcı arabirimi (user interface) oluşturmak için Grafik Aygıt Arabirimi (Graphics Device Interface - GDI) adı verilen bir sistem kullanır. GDI, Windows'ta çizim işlemlerini ve grafik nesneleri oluşturmayı sağlayan bir API'dir. User objeleri de Windows işletim sistemi ile ilgili kullanıcı arabirimi özelliklerini yöneten objelerdir. Bu objeler, kullanıcı arabirimi etkileşimlerinin (pencere yönetimi, menüler, diyalog kutuları, çerçeveler vb.) çoğunu yönetir. User/GDI objeleri, birlikte çalışarak grafik nesneleri oluşturur ve bu nesneleri kullanıcı arayüzüne gösterir. User objeleri, kullanıcı arayüzü olaylarını (event) işler ve GDI aracılığıyla grafik nesneleri oluşturur, yeniden boyutlandırır, döndürür, ölçeklendirir ve çizer. Bu grafik nesneleri, kullanıcı arabirimi bileşenlerini (pencereler, düğmeler, menüler, listeler vb.) oluşturmak için kullanılır. Örneğin, bir uygulama penceresi açmak istediğinde, user objeleri bu işlemi yönetir. Pencerenin başlığı, kenar çubukları, düğmeleri ve diğer bileşenleri GDI aracılığıyla oluşturulur ve kullanıcı arabirimi bileşenleri haline getirilir. Kullanıcı arabirimi bileşenleri oluşturulduktan sonra, user objeleri bu bileşenlerin etkileşimini yönetir.

- Kullanıcı modu uygulamaları tarafından kullanılan nesnelerdir

- Bu nesnelerinin büyük çoğunluğu Windows alt sistemi (Win32k.sys) tarafından kullanılır ve çekirdek ile etkileşime girmez

- GDI objeleri, kullanıcı nesneleri (user objects) olarak kabul edilir ve bu nedenle üçüncü bir kategori olarak sayılabilirler. Ancak, bazı kaynaklar GDI nesnelerini ayrı bir kategori olarak ele alırken, diğerleri "User Objects" kategorisi altında sınıflandırmayı tercih ederler.

Sonuç olarak, user/GDI objeleri, Windows işletim sistemi ile ilgili kullanıcı arabirimi özelliklerini yönetirler ve bu özellikleri kullanarak grafik nesneleri oluştururlar. Bu nesneler, Windows'un kullanıcı arabirimi bileşenlerinin oluşturulmasında kullanılır.

Yürütme nesneleri genellikle bir kullanıcı uygulaması adına bir çevre alt sistemi tarafından veya normal işlemi sırasında işletim sistemi bileşenleri tarafından oluşturulur. Her Windows environment subsystem’ı, uygulamalarına işletim sisteminin farklı bir image’ını sunar. Windows işletim sisteminin iç yapısında yer alan executive objeleri ve object service’leri, environment subsystem’ları tarafından kullanılarak kendi obje ve kaynak sürümlerinin oluşturur. Yani, environment subsystem’ları, bu temel öğeleri kullanarak kendilerine özgü obje ve kaynak sürümleri oluşturabiliyorlar. Bu sayede, farklı environment subsystem’ları farklı kaynakları yönetebilir ve gereksinimlerine uygun objeler oluşturabilirler.

## Kernel Object'leri: Handle İşlemleri

Kernel objelerinin handle’ları process’lere özeldir. Herhangi bir process, varolan bir kernel objesinin ismini ve gerekli erişimleri sağlıyorsa, o object için yeni bir handle oluşturaiblir. 

Her kernel objesi, kendi erişim haklarını barındırır. Örneğin, event handle’lar “set” veya “wait” erişimine (veya her ikisine) sahip olabilir, file handle’lar “read” veya “write” erişimine (veya her ikisine birden) sahip olabilir.

![open_event_handle_diagram_01](/assets/img/2023-03-12-derinlemesine-windows-objects-ve-handles-0x01/open_event_handle_diagram_01.png)

Yukarıdaki diyagramda da gösterildiği gibi “OpenEvent” fonksiyonu kullanılarak ek bir event obje handle’ı uygulama tarafından alınabilir. Ayrıca bu yöntemde iki handle’ın farklı erişim yetkilerinde oluşumu sağlanabilir. Örneğin ilk handle hem set hem de wait’e sahip olurken, diğer oluşan handle sadece wait’e sahip olabilir. Bunu yapan bir C++ kodunu aşağıda inceleyebilirsiniz:

```cpp
#include <Windows.h>
#include <iostream>

int main() {
    HANDLE hEvent = CreateEvent(NULL, TRUE, FALSE, TEXT("MyEvent"));
    if (hEvent == NULL) {
        std::cout << "CreateEvent failed with error code: " << GetLastError() << std::endl;
        return 1;
    }

    HANDLE hEventWaitOnly = OpenEvent(EVENT_MODIFY_STATE | SYNCHRONIZE, FALSE, TEXT("MyEvent"));
    if (hEventWaitOnly == NULL) {
        std::cout << "OpenEvent failed with error code: " << GetLastError() << std::endl;
        return 1;
    }

    std::cout << "Event object successfully created and opened with wait permission" << std::endl;
    return 0;
} 
```

Ek bilgi olarak; File objesi diğer kernel objelerinden biraz farklıdır. Kernel objelerine tek tek ilerde değinilecektir.

## User/GDI Object'leri: Handle İşlemleri

Yazı boyunca user objeleri üzerinde pek durulmayacak fakat yine de yazmaktan zarar gelmez :) Tahmin edileceği üzere kullanıcılar tarafından erişilebilir objelerdir ve user modunda yönetilir.

> Kullanıcı handle’larının process başına varsayılan bir sınırı vardır. Bu sınırı görmek veya editlemek için: HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsNT\CurrentVersion\Windows\USERProcessHandleQuota

Örnek olarak bir window objesi oluşturalım ve daha sonra destroy ederek return edilen handle’ını kapatalım.

```cpp
#include <Windows.h>

LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_CLOSE:
        DestroyWindow(hWnd);
        break;
    default:
        return DefWindowProc(hWnd, msg, wParam, lParam);
    }
    return 0;
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int)
{
    WNDCLASSEX wcex = { sizeof(WNDCLASSEX) };
    wcex.style = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc = WndProc;
    wcex.hInstance = hInstance;
    wcex.hCursor = LoadCursor(NULL, IDC_ARROW);
    wcex.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    wcex.lpszClassName = L"MyWindowClass";
    if (!RegisterClassEx(&wcex))
        return 1;

    HWND hWnd = CreateWindow(L"MyWindowClass", L"My Window", WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, 0, CW_USEDEFAULT, 0, NULL, NULL, hInstance, NULL);
    if (!hWnd)
        return 1;

    ShowWindow(hWnd, SW_SHOWNORMAL);
    UpdateWindow(hWnd);

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    return (int)msg.wParam;
}
```

> Eğer bir console uygulamasında çalışıyorsanız ana fonksiyonun ismini “main” olarak değiştirin ve “WINAPI” belirtecini kaldırın. Fakat C++ Desktop Uygulaması geliştiriyorsanız (bu örnek için önerilir) buna gerek yok.

# Kaynakça ve Referanslar

1. Windows Internals Seventh Edition Part 2 (2022), Microsoft

2. Windows 10 System Programming Part 1 (2020), Leanpub

3. [Understanding SAL \| Microsoft Learn](https://learn.microsoft.com/en-us/cpp/code-quality/understanding-sal?view=msvc-170#sal-basics)

4. [Handleapi.h header - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/#functions)

5. [Object Categories - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/sysinfo/object-categories)

6. [Kernel Objects - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/sysinfo/kernel-objects)

7. [User Objects - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/sysinfo/user-objects)

8. [Securable Objects - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/secauthz/securable-objects)
