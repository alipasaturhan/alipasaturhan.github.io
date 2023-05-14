---
layout: post
title: Windows Portable Executable (PE) Dosyaları
date: 2023-04-12 20:16 +0300
categories: [File Structures, Portable Executable (PE)]
tags: [windows, portable-executable, file-structure]
---

Windows sistemlerinde çalıştırılabilir dosyaların formatıdır. Peki bu ne demek? Belleğe yüklenecek dosya (çalıştırılabilir program) kendi içinde bir düzene sahiptir. Bu düzene PE dosya formatı denir. PE dosya formatı sayesinde PE Loader dosyadaki verileri ayrıştırabilir ve bu bilgileri bir düzen içerisinde belleğe map edebilir.

PE formatı şu dosyaları destekler: .acm .cpl .drv .mui .ocx .scr .sys .tsp .dll .exe 

Aşağıda PE dosya formatının yapısını görebilirsiniz:

![pe_file_preview_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_preview_01.png)

PE formatını destekleyen herhangi bir dosyayı PE View Programı ile analiz ettiğimizde aşağıdaki gibi tree gösterilir:

![pe_file_analyzing_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_01.png)

# Dos Header

Tüm Portable Executable dosyalarında, her bir dosyanın ilk 64 baytını kapsayan bir DOS Header vardır. MS-DOS geliştiricisi ve dosya yapısının mimarı olan Mark Zbikowski anısına MZ kısaltması DOS başlığının ilk başında yer alır.

```cpp
typedef struct _IMAGE_DOS_HEADER {      // DOS .EXE header
    WORD   e_magic;                     // Magic number
    WORD   e_cblp;                      // Bytes on last page of file
    WORD   e_cp;                        // Pages in file
    WORD   e_crlc;                      // Relocations
    WORD   e_cparhdr;                   // Size of header in paragraphs
    WORD   e_minalloc;                  // Minimum extra paragraphs needed
    WORD   e_maxalloc;                  // Maximum extra paragraphs needed
    WORD   e_ss;                        // Initial (relative) SS value
    WORD   e_sp;                        // Initial SP value
    WORD   e_csum;                      // Checksum
    WORD   e_ip;                        // Initial IP value
    WORD   e_cs;                        // Initial (relative) CS value
    WORD   e_lfarlc;                    // File address of relocation table
    WORD   e_ovno;                      // Overlay number
    WORD   e_res[4];                    // Reserved words
    WORD   e_oemid;                     // OEM identifier (for e_oeminfo)
    WORD   e_oeminfo;                   // OEM information; e_oemid specific
    WORD   e_res2[10];                  // Reserved words
    DWORD   e_lfanew;                    // File address of new exe header
  } IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;
```

Yukarıdaki C struct'ı windows internal yapısında winnt.h header'dan bir parçadır. Bu kod PE yapısındaki DOS Header kısmını oluşturur. Bu başlığın tüm üyeleri MS-DOS sistemleri ilgilendirdiği için detaylı bir şekilde incelememize gerek yok. Yapıda bulunan son alan olan e_lfanew dışında;

**e_lfanew:** Eğer PE dosyasının çalıştırıldığı sistem MS-DOS değilse bu üyeye bakılır. Bu üye DOS Stub başlığını atlayarak PE başlığına (NT Header) geçmemizi sağlar. Kısa tanımıyla PE başlığının adresini tutar.

PE View programı tarafından parse edilip düzenli bir şekilde gösterilen DOS Header:

![pe_file_analyzing_dos_header_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_dos_header_01.png)

Üyelerin en sonuncusuna baktığınızda e_lfanew boyutu DWORD (4 BYTE) kadar veri alabilir. Bu üyeye DOS Stub bitişinden hemen sonraki adres yerleştirilecek -> 0x000000F8. İlerleyen konularda buraya tekrar geleceğiz.

Aşağıda PE dosyası DOS başlığının binary halini byte-byte görüyorsunuz:

![pe_file_analyzing_dos_header_binary_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_dos_header_binary_01.png)

# DOS Stub

Eğer bu dosya DOS sisteminde çalıştırılırsa, çalıştırılamayacağını belirtip kapanıyor (DOS Stub'ta verilen mesajı bastırır). Yani bu yapı sadece MS-DOS'un Loader'ı için önemlidir. Başka da bir espirisi yok..

![pe_file_analyzing_dos_stub_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_dos_stub_01.png)

# ~~Rich Header (Undocumented)~~

# NT Header

Bu başlık aslında pe file'ın en önemli iki başlığı barındırır.

```cpp
//0x108 bytes (sizeof)
struct _IMAGE_NT_HEADERS64
{
    ULONG Signature;                                                        //0x0
    struct _IMAGE_FILE_HEADER FileHeader;                                   //0x4
    struct _IMAGE_OPTIONAL_HEADER64 OptionalHeader;                         //0x18
}; 
```

Bununla beraber artık PE dosyasının asıl başlıklarının başladığını belirten Signature kısmı vardır. Bu kısmın değeri de her zaman "PE" dir.

# File Header

Diğer ismi COFF diye bilinir çünkü bu başlık eski windows çalıştırılabilir formatı olarak tanınır.

PE dosya yapısı kendi içinde 64 ve 32 bit olarak ikiye ayrılır. Dolayısıyla yapıda değişiklikler oluşur fakat bunlar o kadar da büyük değişiklikler değildir. Sadece veri tipleri 64 bit'te büyütülmüştür.

Aşağıda gördüğünüz x64 için File Header yapısıdır:

```cpp
//0x14 bytes (sizeof)
struct _IMAGE_FILE_HEADER
{
    USHORT Machine;                                                         //0x0
    USHORT NumberOfSections;                                                //0x2
    ULONG TimeDateStamp;                                                    //0x4
    ULONG PointerToSymbolTable;                                             //0x8
    ULONG NumberOfSymbols;                                                  //0xc
    USHORT SizeOfOptionalHeader;                                            //0x10
    USHORT Characteristics;                                                 //0x12
}; 
```

**Machine:** Çalıştırılabilir dosyanın çalışabileceği işilemci tipini belirten bölümdür.

> PE dosyasının sağladığı işlemci tiplerini görmek için: [PE Format - Win32 apps \| Machine Types](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#machine-types)

**NumberOfSection:** Kaç adet section olduğunu belirten alandır. Teorik olarak bu alanın veri tipine göre 64 bit mimarilerde 65.535 adet section oluşturulabilir. Normal programlarda 4-5 adet section kullanılır.

**TimeDateStamp:** Dosyanın oluşum tarihidir. Burada bulunan değer sistem içerisinde UTC formatına dönüştürülür.

**PointerToSymbolTable:** Sembol tablosunun adresini tutar.

**NumberOfSymbols:** Sembol tablosundaki sembol sayısını tutar.

**SizeOfOptionalHeader:** Optional header'ın boyutunu tutar. Bu kısımdaki veri pe parser'lar için önemli olabilir.

**Characteristics:** PE dosyasının tipini belirten bayraklar'ı tutar. Bu bayraklar pe dosyasının amacını belirtirler.

> PE dosyasının amacını belirten bayrakları görmek için: [PE Format - Win32 apps \| Characteristics Flags](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#characteristics)

Aşağıda bir exe dosyasının File Header kısmı örnek olarak gösterilmiştir.

![pe_file_analyzing_file_header_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_file_header_01.png)

# Optional Header

PE dosyasının en önemli header'ı denilebilir. Bu kısımda da mimari farkından dolayı değişebilen tek önemli detay; 32 bit mimaride BaseOfData (data section'ın başlangıç RVA'sını tutar) bölümünün olmasıdır. 64 bit mimaride bu bölüm kaldırıldı.

Aşağıda x64 Optional Header C++ structure'ını görebilirsiniz:

```cpp
//0xf0 bytes (sizeof)
struct _IMAGE_OPTIONAL_HEADER64
{
    USHORT Magic;                                                           //0x0
    UCHAR MajorLinkerVersion;                                               //0x2
    UCHAR MinorLinkerVersion;                                               //0x3
    ULONG SizeOfCode;                                                       //0x4
    ULONG SizeOfInitializedData;                                            //0x8
    ULONG SizeOfUninitializedData;                                          //0xc
    ULONG AddressOfEntryPoint;                                              //0x10
    ULONG BaseOfCode;                                                       //0x14
    ULONGLONG ImageBase;                                                    //0x18
    ULONG SectionAlignment;                                                 //0x20
    ULONG FileAlignment;                                                    //0x24
    USHORT MajorOperatingSystemVersion;                                     //0x28
    USHORT MinorOperatingSystemVersion;                                     //0x2a
    USHORT MajorImageVersion;                                               //0x2c
    USHORT MinorImageVersion;                                               //0x2e
    USHORT MajorSubsystemVersion;                                           //0x30
    USHORT MinorSubsystemVersion;                                           //0x32
    ULONG Win32VersionValue;                                                //0x34
    ULONG SizeOfImage;                                                      //0x38
    ULONG SizeOfHeaders;                                                    //0x3c
    ULONG CheckSum;                                                         //0x40
    USHORT Subsystem;                                                       //0x44
    USHORT DllCharacteristics;                                              //0x46
    ULONGLONG SizeOfStackReserve;                                           //0x48
    ULONGLONG SizeOfStackCommit;                                            //0x50
    ULONGLONG SizeOfHeapReserve;                                            //0x58
    ULONGLONG SizeOfHeapCommit;                                             //0x60
    ULONG LoaderFlags;                                                      //0x68
    ULONG NumberOfRvaAndSizes;                                              //0x6c
    struct _IMAGE_DATA_DIRECTORY DataDirectory[16];                         //0x70
}; 
```

Bu değerleri olabildiğince kısa tutarak açıklamak gerekirse:

**Magic:** Image'in adresleme yönteminde sınırlandırılma için ayar belirtilir. Magic aşağıdaki değerlerden birisini alabilir.

- PE32 32 bit adresleme yöntemi kullanılır, yani 2 gb limitlidir.

- PE32+ 64 bit adresleme yöntemi kullanır, yani tahmini  8 exabayt.

**~~Major ve Minor'lar:~~** 

**SizeOfCode:** Image'in .text veya komutların barındığı section'ların toplam boyutudur. Section'ın komut taşıyıp taşımadığı ilgili section'ın header'ındaki flag'lerle belirtilir. 

**SizeOfInitializedData:** İlk değeri atanmış (genelde .data section) verilerin bulunduğu section'ların toplam boyutunu tutar. 

**SizeOfUninitializedData:** İlk değeri atanmamış (genelde .bss seciton) verilerin bulunduğu section'ların toplam boyutunu tutar.

**AddressOfEntryPoint:** İlgili image belleğe yüklendikten sonra ilk çalışacak kodun RVA'sı burada tutulur. DLL'ler için bu kısım isteğe bağlıdır. Eğer bir başlangıç kod belirtilmiyorsa bu alan sıfır olmalıdır.

**BaseOfCode:** İlgili image'in belleğe yüklendikten sonraki .text bölümünün RVA'sını tutar.

**BaseOfData:** İlgili image'in belleğe yüklendikten sonraki .data bölümünün RVA'sını tutar. Bu kısım hatırlayacağınız üzere Magic: PE32 için geçerliydi.

**ImageBase:** Image belleğe yüklendiğinde ilk byte'ın tercih edilen adresini tutar. Bu adres 64Kb'ın katları olmalıdır. 

- DLL'ler için varsayılan değer 0x10000000'dir. 

- Windows CE EXE'leri için varsayılan değer 0x00010000'dir. 

- Windows NT, Windows 2000, Windows XP, Windows 95, Windows 98 ve Windows Me için varsayılan değer 0x00400000'dir.

Memory protection servisleri ve diğer birçok nedenlerden dolayı, bu alan tarafından belirtilen adres neredeyse hiç kullanılmaz, bu durumda PE yükleyicisi image'i yüklemek için kullanılmayan bir bellek aralığı seçer. Bu alan seçildikten sonra image bu alana yüklenir ve yeniden yerleştirme (relocation) işlemleri devreye girer. Bu işlem için oluşturulan section'a .reloc deniyor. 

**SectionAlignment:** Image belleğe yükleneceği zaman section'ların bu bölümdeki sayının katlarındaki adreslere yerleşirler. Bu kısım FileAlignment kısmındaki değerden büyük veya eşit olmak zorundadır.

**FileAlignment:** Bu alan diskteki image'in (raw data olarak) section'ların adres katlarını byte olarak içerir. Bu değer 512 - 64Kb arasında 2'nin üssü olmak zorundadır. Ayrıca SectionAlignment değeri mimarinin page boyutundan küçükse, FileAlignment ve SectionAlignment boyutları eşitlenmelidir.

**Win32VersionValue:** Bu kısm reserved ve 0 olması zorunluymuş (bullshit)

**SizeOfImage:** Image'in belleğe yüklendiğinde ne kadar alan kaplayacağını belirtiyor. SectionAlignment'in tuttuğu değerin katlarından olmalıdır.

**SizeOfHeaders:** Dos-Stub, NT Headers (File ve Optional Header'lar) ve Section header'ların toplam boyutudur. FileAlignment'in tuttuğu değerin katlarından olmalıdır.

**CheckSum:** Image file'ın checksum'ıdır. Checksum yapan algoritma IMAGEHELP.dll içerisinde tanımlanmıştır. Belleğe yüklenme sırasında kontrol amaçlı kullanılır.

**Subsystem:** Bu kısım image'in temelde ne olduğunu belirtir. Bir GUI'mi CUI'mi EFI ROM image'i mi gibi değerlerden birini taşır. Kısaca image'in çalıştırılması için gereken Windows alt sistemini belirtir.

> PE dosyasının tipini belirten değerleri görmek için: [PE Format - Win32 apps \| Sub-Systems](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#windows-subsystem)

**DllCharacteristics:** DLL hakkındaki özellikleri belirtir.

> Bu alana gelebilecek değerleri görmek için: [PE Format - Win32 apps \| Dll-Characteristics](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#dll-characteristics)

**SizeOfStackReserve:** Program için ayrılacak Stack boyutu. Genelde 1MB (0x10000) değerini içerir. Eğer CreateThread API’si kullanılarak oluşturulan threadlere stack boyutu verilmezse, oluşturulacak thread için bu alandaki değer geçerli olur. Burada belirtilen alanın tamamı hemen threade verilmez, verilecek alan SizeOfStackCommit'in tuttuğu değer kadardır.

**SizeOfStackCommit:** Program başlangıcında commit edilecek stack boyutudur. SizeOfStackReserve'in katları olmalıdır. Genelde bu alanın değeri 4Kb (0x1000) olur.

**SizeOfHeapReserve:** Program için ayrılacak olan Heap boyutudur. Genelde 0x10000 değerindedir fakat Heap dinamik işlemler için kullanıldığından bu boyut runtime sırasında artabilir.

**SizeOfHeapCommit:** Program başlangıcında commit edilecek heap boyutudur. Genelde 0x1000 değerindedir. Bu alan da SizeOfHeapReserve'in katlarında olmalıdır.

**LoaderFlags:** Bu kısm da reserved ve 0 olması zorunluymuş (bullshit2)

**NumberOfRvaAndSizes:** Data Directory yapısının boyutunu tutar.

Aşağıda bir exe dosyasının Optional Header kısmı örnek olarak gösterilmiştir.

![pe_file_analyzing_optional_header_binary_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_optional_header_binary_01.png)

> Bir sonraki başlığa giriş yapmadan önce küçük ama çok önemli bir konuya değineyim:
> 
> VA istediğin datanın tam adresidir. Hesaplaması: VA = ImageBase + RVA
> 
> RVA, imagebase'den istenen dataya olan uzaklıktır. Hesaplaması: RVA = VA - ImageBase
> 
> Bu ikisi PE için hayati öneme sahip kavramlardır.

# PE Dosyasının Memory'de Analizi

Aşağıda memory'e yüklenmiş olan image'in optional header'ına kadar ulaşılmıştır:

```cpp
lkd> !process 0 0 notepad.exe
PROCESS ffff83859107a080
    SessionId: 8  Cid: 3634    Peb: dfea594000  ParentCid: 3254
    DirBase: 17e641002  ObjectTable: ffffc880d85d6d80  HandleCount: 239.
    Image: notepad.exe
lkd> dt nt!_PEB dfea594000 -a Image*
   +0x003 ImageUsesLargePages : 0y0
   +0x010 ImageBaseAddress : 0x00007ff6`69630000 Void
   +0x128 ImageSubsystem : 2
   +0x12c ImageSubsystemMajorVersion : 0xa
   +0x130 ImageSubsystemMinorVersion : 0
lkd> dt nt!_IMAGE_DOS_HEADER 00007ff669630000
   +0x000 e_magic          : 0x5a4d
            ...
            ...
   +0x03c e_lfanew         : 0n256
lkd> dt nt!_IMAGE_NT_HEADERS64 00007ff669630000+0n256
   +0x000 Signature        : 0x4550
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER64
lkd> dt nt!_IMAGE_FILE_HEADER 00007ff669630000+0n256+4
   +0x000 Machine          : 0x8664
   +0x002 NumberOfSections : 7
   +0x004 TimeDateStamp    : 0xbdd4adcd
   +0x008 PointerToSymbolTable : 0
   +0x00c NumberOfSymbols  : 0
   +0x010 SizeOfOptionalHeader : 0xf0
   +0x012 Characteristics  : 0x22
lkd> dt nt!_IMAGE_OPTIONAL_HEADER64 00007ff669630000+0n256+18
   +0x000 Magic            : 0x20b
   +0x002 MajorLinkerVersion : 0xe ''
   +0x003 MinorLinkerVersion : 0x14 ''
   +0x004 SizeOfCode       : 0x24800
   +0x008 SizeOfInitializedData : 0xe000
   +0x00c SizeOfUninitializedData : 0
   +0x010 AddressOfEntryPoint : 0x23f40
   +0x014 BaseOfCode       : 0x1000
   +0x018 ImageBase        : 0x00007ff6`69630000
   +0x020 SectionAlignment : 0x1000
   +0x024 FileAlignment    : 0x200
   +0x028 MajorOperatingSystemVersion : 0xa
   +0x02a MinorOperatingSystemVersion : 0
   +0x02c MajorImageVersion : 0xa
   +0x02e MinorImageVersion : 0
   +0x030 MajorSubsystemVersion : 0xa
   +0x032 MinorSubsystemVersion : 0
   +0x034 Win32VersionValue : 0
   +0x038 SizeOfImage      : 0x38000
   +0x03c SizeOfHeaders    : 0x400
   +0x040 CheckSum         : 0x31761
   +0x044 Subsystem        : 2
   +0x046 DllCharacteristics : 0xc160
   +0x048 SizeOfStackReserve : 0x80000
   +0x050 SizeOfStackCommit : 0x11000
   +0x058 SizeOfHeapReserve : 0x100000
   +0x060 SizeOfHeapCommit : 0x1000
   +0x068 LoaderFlags      : 0
   +0x06c NumberOfRvaAndSizes : 0x10
   +0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY
```

görüldüğü gibi ImageBase _PEB yapısındaki ImageBaseAddress ile aynıdır ve diskteki pe image'i incelediğimizde bu alanın default olarak 0x00400000 olduğunu görüyorduk. Fakat önceden de belirtildiği bu adrese bellek koruma mekanizmaları ve diğer sebeplerden (relocation vs.) ötürü image, bellekte bu adrese yerleşemiyor.

Şimdi Data Directory kısmına göz atalım. Önceki yazıda data directory kısmını hatırlarsanız ilk index Export, ikinci index Import diye gidiyordu. Biz notepad.exe dosyasına baktığımız için bu dosyanın bir şey export etmesini beklemeyiz. Fakat yine de hızlıca bu directory'lere bakalım ve emin olalım:

![notepad_memory_analysis_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/notepad_memory_analysis_02.png)

İlk olarak export directory'i getirelim:

```cpp
lkd> dt nt!_IMAGE_DATA_DIRECTORY 00007ff669630000+0n256+18+70
   +0x000 VirtualAddress   : 0
   +0x004 Size             : 0
```

Şimdi import directory'i getirelim:

```cpp
lkd> dt nt!_IMAGE_DATA_DIRECTORY 00007ff669630000+0n256+18+70+8
   +0x000 VirtualAddress   : 0x2d0c0
   +0x004 Size             : 0x244
```

> Bu analiz sadece bir örnekti. İlerideki başlıklar için windbg kullanımına biraz da olsa aşina olmalısınız. 

# Data Directory

Optional Header C++ structure'ının en altına baktığınızda bu bölümün 16 index'ten oluşan bir dizi olduğunu görmüşsünüzdür. Data directory image'in belleğe yüklenirkenki kullanılan önemli bilgileri barındırır.

Gelin tekrar optional header'ın en altındaki yapıya bakalım:

```cpp
//0xf0 bytes (sizeof)
struct _IMAGE_OPTIONAL_HEADER64
{
    USHORT Magic;                                                           //0x0
    ...                                 
    struct _IMAGE_DATA_DIRECTORY DataDirectory[16];                         //0x70
}; 
```

Ve aslında winnt.h header'ı altında şu şekilde saklanır:

```cpp
#define IMAGE_NUMBEROF_DIRECTORY_ENTRIES 16
IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
```

Yani her bir data directory index'i aşağıdaki gibi yapıdan oluşur:

```cpp
//0x8 bytes (sizeof)
struct _IMAGE_DATA_DIRECTORY
{
    ULONG VirtualAddress;                                                   //0x0
    ULONG Size;                                                             //0x4
}; 
```

**VirtualAddress:** Belirtilen data dizininin RVA'sını tutar.

**Size:** İlgili data dizininin boyutunu tutar.

Her bir index değeri aşağıdaki gibi define edilmiştir ve her birisinin VirtualAddress ve Size bölümü vardır:

```cpp
#define    IMAGE_DIRECTORY_ENTRY_EXPORT        0
#define    IMAGE_DIRECTORY_ENTRY_IMPORT        1
#define    IMAGE_DIRECTORY_ENTRY_RESOURCE        2
#define    IMAGE_DIRECTORY_ENTRY_EXCEPTION        3
#define    IMAGE_DIRECTORY_ENTRY_SECURITY        4
#define    IMAGE_DIRECTORY_ENTRY_BASERELOC        5
#define    IMAGE_DIRECTORY_ENTRY_DEBUG        6
#define    IMAGE_DIRECTORY_ENTRY_COPYRIGHT        7
#define    IMAGE_DIRECTORY_ENTRY_GLOBALPTR        8   /* (MIPS GP) */
#define    IMAGE_DIRECTORY_ENTRY_TLS        9
#define    IMAGE_DIRECTORY_ENTRY_LOAD_CONFIG    10
#define    IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT    11
#define    IMAGE_DIRECTORY_ENTRY_IAT        12  /* Import Address Table */
#define    IMAGE_DIRECTORY_ENTRY_DELAY_IMPORT    13
#define    IMAGE_DIRECTORY_ENTRY_COM_DESCRIPTOR    14
```

PE dosyasında bu data dizinleri aşağıdaki gibi görüntülenmektedir:

![pe_file_analyzing_data_directory_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_data_directory_01.png)

Yukarıdaki gibi değeri 0 olan data dizinleri görmezden gelinir, yani herhangi bir veri yok anlamına gelir.

# Section Headers

PE yapısına sahip dosyadaki her sectionın kendisine ait bir header'ı vardır. Bu header temsil ettiği section hakkında önemli bilgiler verir. Örneğin bu section bir kod section'ı mı yoksa data section'ı mı, veya pe dosyasındaki resource'ları mı tutuyor vb. gibi bilgiler saklar.

Aşağıda her section'a ait header'ı belirten c++ structure'ını görebilirsiniz:

```cpp
#define IMAGE_SIZEOF_SHORT_NAME 8

typedef struct _IMAGE_SECTION_HEADER {
  BYTE  Name[IMAGE_SIZEOF_SHORT_NAME];
  union {
    DWORD PhysicalAddress;
    DWORD VirtualSize;
  } Misc;
  DWORD VirtualAddress;
  DWORD SizeOfRawData;
  DWORD PointerToRawData;
  DWORD PointerToRelocations;
  DWORD PointerToLinenumbers;
  WORD  NumberOfRelocations;
  WORD  NumberOfLinenumbers;
  DWORD Characteristics;
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
```

Bu structure'daki bölümleri açıklamak gerekirse:

**Name:** İlgili section'ın ismini belirtir (UTF-8 destekler) ve section adları 8 karakterden uzun olmaz.

**PhysicalAddress ve VirtualSize:** Section'ın belleğe yüklendiğinde bellekte ne kadar yer kaplayacağını belirtir.

**VirtualAddress:** PE file memory'e yükleneceği zaman ilgili section'ın ilk byte'ının memory için RVA'sını tutar. Optional Header içerisindeki SectionAlignment değerinin katlarından olmalıdır.

**SizeOfRawData:** Section'ın diskteki boyutunu içerir ve  Optional Header'daki FileAlignment'in değerine göre yuvarlanır (yani katlarından olmalıdır).

**PointerToRawData:** PE dosyasında (diskteki) ilgili section'ın pointer'ıdır. Burası da tahmin edeceğiniz üzere FileAlignment içerisindeki değerinin katlarındandır.

**PointerToRelocations:** Bu kısım exe dosyaları için 0 set edilir.

**PointerToLinenumbers:** Bu kısım exe dosyaları için 0 set edilir.

**NumberOfRelocations:** Bu kısım exe dosyaları için 0 set edilir.

**NumberOfLinenumbers:** Bu kısım kullanımdan kaldırıldığı için genelde 0 set edilir.

**Characteristics:** Section'ın ve section içerisindeki datanın özelliklerini belirten bayraklar tutar.

> Bu alana gelebilecek değerleri görmek için: [PE Format - Win32 apps \| Section Flags](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format?redirectedfrom=MSDN#section-flags)

Genel olarak section header'da üç temel alan ele alınıyor. Biri image'in bellekteki adresi ve boyutu, biri diskteki adresi ve boyutudur, bir diğeri de Characteristics alanı.

![pe_file_section_header_structure_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_section_header_structure_01.png)

Aşağıda .text yani genelde dosyadaki kodların bulunduğu kısımların header'ı görebilirsiniz:

![pe_file_analyzing_text_section_header_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_text_section_header_01.png)

Burada da .data (ilk değeri verilmiş değişkenler) kısmının header'ını görebilirsiniz:

![pe_file_analyzing_data_section_header_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_file_analyzing_data_section_header_01.png)

Burada değinilmesi gerekilen önemli bir konu vardır.

SizeOfRawData ve VirtualSize farklı olabilir ve bunun birden fazla sebebi olabilir.

SizeOfRawData, IMAGE_OPTIONAL_HEADER.FileAlignment'ın katı olmalıdır, bu nedenle bölüm boyutu bu değerden küçükse geri kalanı dolgu yapılır ve SizeOfRawData, IMAGE_OPTIONAL_HEADER.FileAlignment'ın en yakın katına yuvarlanır.
Ancak bölüm belleğe yüklendiğinde bu hizalamayı takip etmez ve yalnızca bölümün gerçek boyutu işgal edilir.
Bu durumda SizeOfRawData, VirtualSize'dan daha büyük olacaktır.

Tersi de olabilir.
Bölüm başlatılmamış veri içeriyorsa, bu veriler diskte hesaba katılmaz, ancak bölüm belleğe eşlendiğinde, bölüm daha sonra başlatılan ve kullanılan başlatılmamış veriler için bellek alanı ayırmak için genişleyecektir.
Bu, bölümün disktedeki boyutunun, bellekteki boyutundan daha az işgal edeceği anlamına gelir, bu durumda VirtualSize, SizeOfRawData'dan daha büyük olacaktır.

# Sections

Tüm bölümler, RAM'e yüklendiğinde 'SectionAlignment' hizalanır ve dosyada 'FileAlignment' hizalanır. Bölümler, bölüm başlıklarındaki girdilerle tanımlanır: Bölümleri dosyada 'PointerToRawData' ve bellekte 'VirtualAddress' ile bulursunuz; uzunluk 'SizeOfRawData'da belirtilir.

Bunların içerdiklerine bağlı olarak çeşitli türde bölümler vardır. Çoğu durumda (ancak her zaman değil) isteğe bağlı başlık veri dizisinde bir işaretçi ile bir veri dizini içeren en az bir bölüm olacaktır.

~~Burada tanımlanan tüm bölümler PE Optional başlığında belirtilen SectionAlignment bilgisine göre hafızada, FileAlignment bilgisine göre ise dosyada hizalanıyor.~~

- **.text** → the actual code the binary runs
- **.data** → read/write data (globals)
- **.rdata** → read-only data (strings)
- **.bss** → Block Storage Segment (uninitialzed data format), often merged with the .data section
- **.idata** → import address table, often merged with .text or .rdata sections
- **.edata** → export address table
- **.pdata** → some architectures like ARM, MIPS use these sections structures to aid in stack-walking at run-time
- **PAGE*** → code/data which it’s fine to page out to disk if you’re running out of memory
- **.reolc** → relocation information for where to modify the hardcoded addresses
- **.rsrc** → resources like icons, other embedded binaries, this section has a structure organizing it like a filesystem

> Bu alana gelebilecek değerleri görebilmek için: [PE Format - Win32 apps \| Special Sections](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#special-sections)

# PE: Import

Image'ler (pe dosyaları) kod section'larında sadece kendi fonksiyonlarını kullanmazlar. Windows'un sağladığı APı fonksiyonlarını kullanırlar. Bu fonksiyonları kullanabilmeleri için ise kendi section'larında hangi fonksiyonu kullandığını, kullandığı fonksiyon hangi library içerisinde, fonksiyon adresi gibi gibi bilgileri saklaması gerekir. İşte burada Import dediğimiz kavram devreye girer.

PE dosyasında import edilen fonksiyonlar, library'ler image'in belirli bölgelerinde detaylı şekilde belirtilir. Program compile edilip komutların makine koduna dönüştürüldükten sonra ikinci aşama olan linkleme (linker ile) işlemine sokulur. Linker ilgili programın kullandığı fonksiyon/library'lerin adreslerini çözümleyerek programa bağlar ve artık hazır olan programımız external dll'deki fonksiyonları call edebilir.

> *PE: Import* başlığını yazmayı bitirdikten sonra bu notu yazıyorum; aşağıdaki diyagram birazdan okuyacağınız yazıların şematize edilmiş halidir. İlk baktığınızda bir şey anlamamanız normal, fakat alttaki yazıları okuyarak bu görsele tekrar tekrar bakın. PE'deki import mekanizması bu görselde açıklanmıştır.

![pe_import_mechanism_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_import_mechanism_01.png)

PE file'da data directory'leri tutan bir array olduğunu belirtmiştik. Bu array'de import işlemini ilgilendiren index 1. sidir (0. olan Export, ona da gelicez). Burada iki adet bölüm olduğunu hatırlamışsınızdır.

- VirtualAddress: IMAGE_IMPORT_DESCRIPTOR yapısının RVA'sını tutar.

- Size: Bu da ilgili data dizininin boyutunu veriyordu.

IMAGE_IMPORT_DESCRIPTOR yapısı PE dosyasının import ettiği tek bi DLL'i tanımlayan art arda gelen yapılardır. Örneğin 3 adet DLL import edilmişse Import Directory Table'da bulunan RVA ilk DLL i temsil eden IMAGE_IMPORT_DESCRIPTOR yapısına uzanır ve 3 adet IMAGE_IMPORT_DESCRIPTOR oluşturulması gerekir.

Her bir IMAGE_IMPORT_DESCRIPTOR aşağıdaki yapıdan oluşur:

```cpp
typedef struct _IMAGE_IMPORT_DESCRIPTOR
{
    union {
        DWORD Characteristics; // 0 for terminating null import descriptor
        DWORD OriginalFirstThunk; // RVA to original unbound IAT (PIMAGE_THUNK_DATA)
    } DUMMYUNIONNAME;

    DWORD TimeDateStamp; // 0 if not bound,
                         // -1 if bound, and real date    ime stamp
                         // in IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT (new BIND)
                         // O.W. date/time stamp of DLL bound to (Old BIND)

    DWORD ForwarderChain; // -1 if no forwarders
    DWORD Name;
    DWORD FirstThunk; // RVA to IAT (if bound this IAT has actual addresses)
} IMAGE_IMPORT_DESCRIPTOR;
```

**OriginalFirstThunk (OFT):** PE dosyasındaki Import Lookup Table (ILT) / Import Name Table (INT) yapısnın RVA'sını tutar. Burada ilgili dll tarafından import edilen function'ların ascii karşılıklarını tutan RVA vardır. Yani aslında biraz sonra göreceğiniz IMAGE_THUNK_DATA yapısının AddressOfData bölümünü temsil eder. 

IMAGE_IMPORT_DESCRIPTOR bölümü açıklandıktan sonra bu iki yapı da açıklanacaktır.

> Import Lookup Table (ILT) diğer ismi Import Name Table/Array diye de geçer. İkisi de aynı kavramdır. Yani IMAGE_THUNK_DATA yapısı bu kavramları temsil eder. ILT=INT

**TimeDateStamp:** Buradaki değer 0 ise import edilen dll bound değildir. Fakat 0 harici bir değerde ise bound import edilmiştir.

**ForwarderChain (FC):** Bu alan DLL forwarding'den sorumludur. 

> DLL yönlendirmesi: Bir DLL'nin bazı import edilen fonksiyonlarını başka bir DLL'ye yönlendirdiği durumdur.

**Name:** Import edilen DLL'in adının ASCII karakterlerini içeren bir dize tablosuna olan RVA'sını tutar.

**FirstThunk (FT):** PE dosyasındaki Import Address Table (IAT) yapısnın RVA'sını tutar. Buradaki RVA, ilgili DLL dosyasındaki ilk import edilen fonksiyonun adresine götürür. Bu yapı da seriler halinde devam eder. Yani IMAGE_IMPORT_DESCRIPTOR'ın temsil ettiği DLL'in fonksiyonları için tek tek bu yapıdan oluşturulur.

*INT (ILT), IAT ile aynı olan IMAGE_THUNK_DATA yapısının bir dizisidir. Her ikisi de aynı structure'ı işaret ettiğinden, IAT ve INT arasındaki temel fark, INT'nin yürütülebilir belleğe yüklendiğinde Windows yükleyicisi tarafından üzerine yazılmaması, ancak IAT girişlerinin aktarılan işlevin gerçek adresiyle üzerine yazılmasıdır.*

*Ayrıca, INT yürütülebilir dosyanın yüklenmesi için gerekli değildir ancak IAT, bir yürütülebilir dosyanın yüklenmesi için temel bileşenlerden biridir. Bu olmadan, yürütülebilir dosya yüklenmeyebilir. IAT'deki fonknsiyon adresleri image belleğe yüklenirken doldurulur. Yani diskteki bir PE için IAT tablosunda geçerli bir değer bulamıyoruz, buradaki değerler linker için de bir adres ifade etmiyor (bu başlıktan sonra göreceğimiz bound importlama işlemi hariç).*

Buraya kadar anlatılanları şematize etmek istersek:

![pe_import_mechanism_on_disk_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_import_mechanism_on_disk_01.png)

Nedir bu IMAGE_THUNK_DATA? Bu yapı ise ilgili fonksiyonla ilgili bilgileri tutar. Örneğin fonksiyonun ismini belirten ascii dizisine olan RVA'yı, fonksiyonun IAT'de gösterilen VA'sı, ordinal mi değil mi (fonksiyonun nasıl call edildiği) gibi bilgileri kapsar.

Aşağıda IMAGE_THUNK_DATA yapısını görebilirsiniz:

```cpp
typedef struct _IMAGE_THUNK_DATA64 {
    union {
        ULONGLONG ForwarderString;
        ULONGLONG Function;
        ULONGLONG Ordinal;
        ULONGLONG AddressOfData;
    } u1;
} IMAGE_THUNK_DATA64,*PIMAGE_THUNK_DATA64;

typedef struct _IMAGE_THUNK_DATA32 {
    union {
        DWORD ForwarderString;
        DWORD Function;
        DWORD Ordinal;
        DWORD AddressOfData;
    } u1;
} IMAGE_THUNK_DATA32,*PIMAGE_THUNK_DATA32;
```

**Function:** IMAGE_IMPORT_DESCRIPTOR yapısındaki FirstThunk bölümünün aslında buradaki veriye işaret eden RVA'yı tutar demiştik. Buradaki değer PE'ye import edilecek function'ın VA'sını (Virtual Address) barındırır.

**AddressOfData:** Bu bölümde ise IMAGE_IMPORT_DESCRIPTOR yapısındaki OriginalFirstThunk bölümü aslında AddressOfData bölümünün RVA'sını tutuyor demiştik. Ve bu RVA ise IMAGE_IMPORT_BY_NAME yapısına uzanır.

IMAGE_IMPORT_BY_NAME bölümü her fonksiyon için Hint/Name adında iki bölümü tanıtır.

```cpp
typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD    Hint;
    BYTE    Name[1];
} IMAGE_IMPORT_BY_NAME,*PIMAGE_IMPORT_BY_NAME;
```

**Hint:** AddressOfData indeksini tutar. Burayı PE'ye dahil edilen fonksiyon havuzundan kaçıncı fonksiyon gibi düşünebilirsiniz.

**Name:** İlgili fonksiyonun ascii dizisini tutar. Yani burada fonksiyonun ismi gözükür.

![pe_import_mechanism_02.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_import_mechanism_02.png)

Biliyorum çok fazla sözel gittik ve kafanız karışmış olabilir, şimdi size bu anlattıklarımı bide uygulamalı olarak göstereyim. Fakat uygulamaya geçmeden önce tekrardan yukarıdaki diyagramlara bakmanızı tavsiye ederim:

Aşağıda her dll dosyasını temsil eden IMAGE_IMPORT_DESCRIPTOR yapısının import edilen fonksiyonları gösteriliyor. İlgili process'in PEB yapısındaki ImageBase bölümünde tutulan adresi bulduktan sonra aşağıdaki gibi PE dosyasının Import Data Directory bölümüne ulaşabilirsiniz:

```cpp
lkd> dt nt!_IMAGE_DATA_DIRECTORY 00007ff68d0c0000+0n256+18+70+8
   +0x000 VirtualAddress   : 0x2d0c0
   +0x004 Size             : 0x244
lkd> dc /c5 00007ff68d0c0000+2d0c0
00007ff6`8d0ed0c0  0002d3d8 00000000 00000000 0002e1f0 000268b8  .................h..
00007ff6`8d0ed0d4  0002d320 00000000 00000000 0002e342 00026800   ...........B....h..
00007ff6`8d0ed0e8  0002d670 00000000 00000000 0002e7d2 00026b50  p...............Pk..
00007ff6`8d0ed0fc  0002dba8 00000000 00000000 0002e83a 00027088  ............:....p..
00007ff6`8d0ed110  0002db80 00000000 00000000 0002e85c 00027060  ............\...`p..
00007ff6`8d0ed124  0002da60 00000000 00000000 0002eb16 00026f40  `...............@o..
00007ff6`8d0ed138  0002d8a0 00000000                             ........44
```

Burada gördüğünüz 20 byte'lık satırların her birisi bir adet IMAGE_IMPORT_DESCRIPTOR demektir. Aşağıda ilk IMAGE_IMPORT_DESCRIPTOR yapısının değerleri belirtilmiştir:

- (4-byte) OriginalFirstThunk: 0002D3D8

- (4-byte) TimeDateStamp: 00000000

- (4-byte) ForwarderChain: 00000000

- (4-byte) Name: 0002E1F0

- (4-byte) FirstThunk: 000268B8

> Eğer bu verileri direkt olarak memory viewer ile bakarsanız dikkat edin. Bilgisayarınız little endian ise bellekte byte'ların dizilimi ters yazılır. Örneğin bellekte 3A 5B gibi bir veriniz varsa bu 5B 3A diye okunur, B5 A3 diye değil.

Önce bu yapı hangi DLL'i temsil ediyormuş ona bakalım:

```cpp
lkd> da 00007ff68d0c0000+0002e1f0
00007ff6`8d0ee1f0  "KERNEL32.dll"
```

İlk DLL kernel32.dll olarak gözüküyor. Şimdi bu dll'deki import edilen fonksiyonları IAT'de incelemek istersek FirstThunk RVA'sını aşağıdaki gibi kullanabiliriz: 

```cpp
lkd> dps 00007ff68d0c0000+000268b8
00007ff6`8d0e68b8  00007ffe`9c2bb630
00007ff6`8d0e68c0  00007ffe`9c2c50f0
00007ff6`8d0e68c8  00007ffe`9d331760
00007ff6`8d0e68d0  00007ffe`9d320fc0
00007ff6`8d0e68d8  00007ffe`9c2c4ff0
00007ff6`8d0e68e0  00007ffe`9c2b6180
00007ff6`8d0e68e8  00007ffe`9c2bd890
00007ff6`8d0e68f0  00007ffe`9c2db180
00007ff6`8d0e68f8  00007ffe`9c2c0910
00007ff6`8d0e6900  00007ffe`9c2b6120
00007ff6`8d0e6908  00007ffe`9c2c0580
00007ff6`8d0e6910  00007ffe`9c2c52c0
00007ff6`8d0e6918  00007ffe`9c2c5640
00007ff6`8d0e6920  00007ffe`9c2c5760
00007ff6`8d0e6928  00007ffe`9c2c4fe0
00007ff6`8d0e6930  00007ffe`9c2bfb20
```

İşte buradaki adresler IAT'yi kapsar, yani her fonksiyonun gerçek adresi buradadır. 

Bu fonksiyonun ismine ulaşmak istediğimizde ise OriginalFirstThunk bölümünü ele almalıyız. Bu bölüm de hatırlayacağınız üzere AddressOfData bölümündeki RVA'yı temsil ediyordu. Aşağıda OriginalFirstThunk bölümünü getirdiğimizde:

```cpp
lkd> dps /c1 00007ff68d0c0000+0002d3d8
00007ff6`8d0ed3d8  00000000`0002de34
00007ff6`8d0ed3e0  00000000`0002de46
00007ff6`8d0ed3e8  00000000`0002de58
00007ff6`8d0ed3f0  00000000`0002de70
00007ff6`8d0ed3f8  00000000`0002de88
00007ff6`8d0ed400  00000000`0002de9e
00007ff6`8d0ed408  00000000`0002deb0
00007ff6`8d0ed410  00000000`0002dec4
00007ff6`8d0ed418  00000000`0002ded2
00007ff6`8d0ed420  00000000`0002dee6
00007ff6`8d0ed428  00000000`0002def4
00007ff6`8d0ed430  00000000`0002df06
00007ff6`8d0ed438  00000000`0002df14
00007ff6`8d0ed440  00000000`0002df20
00007ff6`8d0ed448  00000000`0002df2a
00007ff6`8d0ed450  00000000`0002df3e
```

RVA'larıyla karşılaşıyoruz. Buradaki ilk RVA bizim üzerinde ilerlediğimiz fonksiyonun RVA'sıdır. Şimdi bu RVA sayesinde fonksiyonun ismini tutan VA'ya ulaşabiliriz:

```cpp
lkd> dc 00007ff68d0c0000+00000000`0002de34
00007ff6`8d0ede34  654702b8 6f725074 64644163 73736572  ..GetProcAddress
00007ff6`8d0ede44  00dc0000 61657243 754d6574 45786574  ....CreateMutexE
00007ff6`8d0ede54  00005778 63410001 72697571 57525365  xW....AcquireSRW
00007ff6`8d0ede64  6b636f4c 72616853 00006465 65440114  LockShared....De
00007ff6`8d0ede74  6574656c 74697243 6c616369 74636553  leteCriticalSect
00007ff6`8d0ede84  006e6f69 65470221 72754374 746e6572  ion.!.GetCurrent
00007ff6`8d0ede94  636f7250 49737365 02be0064 50746547  ProcessId...GetP
00007ff6`8d0edea4  65636f72 65487373 00007061 65470281  rocessHeap....Ge"
```

Burası AddressOfData bölümünün işaret ettiği yer olduğu için IMAGE_IMPORT_BY_NAME yapısına ulaştığımızı varsayabiliriz. Yine ilk verilere baktığınızda:  65 47 02 B8 6F 72 50 74 64 64 41 63 73 73

- (2-byte) Hint: 02 B8

- (1-byte) Name: 47 65 74 50 72 6F 63 41 64 64 73 73

Daha basit yoldan bu değerlere ulaşmak isterseniz XNTSV uygulamasını kullanabilirsiniz:

![notepad_import_analysis_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/notepad_import_analysis_01.png)

> Process Environment Block (PEB) user space'te bulunduğu için XNTSV ile Kernel modunda geçmemize gerek yoktur.

## Bound Import

PE dosyaları için farklı bir import'lama işlemi gerçekleştirir. Burada ise RVA'larla uğraşmak yerine direkt olarak fonksiyonun gerçek adresi devreye girer. Bu yöntem kullanıldığı taktirde eğer fonksiyonun buluduğu DLL'de bir değişiklik olursa, bound import edilen fonksiyonun gerçek adresi kayabilir ve programın crash olabilme imkanı vardır.

DLL'in bound olup olmamasını görebilmek için önceden de beliritlen IMAGE_IMPORT_DESCRIPTOR yapısındaki TimeDateStamp yapısına bakılır. Bu değer 0 olduğunda normal importlama işlemi gerçekleşmiş demektir, 0 harici bir değer ise fonksiyonun adresleri fixed'dir. Bound import'lama işlemi gerçekleştiğinde ise aşağıdaki structure devreye giriyor:

```cpp
typedef struct _IMAGE_BOUND_IMPORT_DESCRIPTOR
{
    DWORD   TimeDateStamp;
    WORD    OffsetModuleName;
    WORD    NumberOfModuleForwarderRefs;
/* Array of zero or more IMAGE_BOUND_FORWARDER_REF follows */
} IMAGE_BOUND_IMPORT_DESCRIPTOR,  *PIMAGE_BOUND_IMPORT_DESCRIPTOR;TOR;
```

IAT'de bound import edilen DLL'ler için yukarıdaki structure yanyana oluşur.

Bu yapıdaki NumberOfModuleForwarderRefs bölümü ise başka bir yapıyı işaret eder.

```cpp
typedef struct _IMAGE_BOUND_FORWARDER_REF
{
    DWORD   TimeDateStamp;
    WORD    OffsetModuleName;
    WORD    Reserved;
} IMAGE_BOUND_FORWARDER_REF, *PIMAGE_BOUND_FORWARDER_REF;
```

Bu yapı ise ilgili DLL'deki forward edilmiş fonksiyonları temsil eder.

## Delay Import

İsimden de anlaşılacağı gibi import mekanizması diğer DLL dosyalarını import etmek için gecikiyor. Neden gecikiyor? Çünkü bazı import edilen fonksiyonlar daha az sıklıkla veya sadece belirli bölümlerde kullanılabilir ve program bir süre çalıştıktan sonra bazı fonksiyonlar kullanılabilir hale gelebilir. Yani bu fonksiyonlar program belleğe yüklenirken hemen başlatılması gerekmez.

Delay import'lamayı temsil eden bir tablo vardır/dizi vardır. Bu yapı da Import Table gibi her Delay Import'lanan DLL bir _IMAGE_DELAYLOAD_DESCRIPTOR yapısına sahiptir ve bellekte bu yapılar sıralı haldedir. 

```cpp
typedef struct _IMAGE_DELAYLOAD_DESCRIPTOR
{
    union
    {
        DWORD AllAttributes;
        struct
        {
            DWORD RvaBased:1;
            DWORD ReservedAttributes:31;
        } DUMMYSTRUCTNAME;
    } Attributes;

    DWORD DllNameRVA;
    DWORD ModuleHandleRVA;
    DWORD ImportAddressTableRVA;
    DWORD ImportNameTableRVA;
    DWORD BoundImportAddressTableRVA;
    DWORD UnloadInformationTableRVA;
    DWORD TimeDateStamp;
} IMAGE_DELAYLOAD_DESCRIPTOR, *PIMAGE_DELAYLOAD_DESCRIPTOR;
```

**AllAttributes:** Sürümleri ayırt etmek için kullanılır. 

- Yeni sürümü belirtmek için: 1

- Eski sürümü belirtmek için: 0

Yeni sürümde RVA kullanılır, eski sürümde ise pointer'lar kullanılır. Genelde yeni sürüm ele alınır.

**DllNameRVA:** DLL'in ismini gösteren RVA'yı tutar.

**ModuleHandleRVA:** Import edilen DLL'nin başlangıç adresine işaret eden bir RVA.

**ImportAddressTableRVA:** İlgili DLL import edildiğinde IAT'deki gerçek RVA'sını tutan bölümüdür.

**ImportNameTableRVA:** Önceden anlatılan _IMAGE_IMPORT_BY_NAME yapısına ilgili fonksiyon için RVA tutar. Bu RVA'yı da takip ettiğinizde ilgili fonksiyon sanki normal import metodu kullanılıyormuş gibi düşünebilirsiniz.

**BoundImportAddressTableRVA:** Bu fonksiyonun IAT'deki bound (fixed adresi) halinin RVA'sını tutar. Optional olarak görülür. Çünkü ilgili RVA'yı takip ettiğinizde genelde bir veri yoktur.

~~**UnloadInformationTableRVA:** Bu tablo Delay Import'taki DLL'de yüklenmemiş fonksiyonunu belirtir.~~

~~**TimeDateStamp:** Delay Import'lanan DLL'in timestamp'ini tutar.~~

Ayrıca bu yapıdaki bölümler yine bazı structure'ları işaret eder. Yine mi... dediğinizi duya gibiyim :) ama bu bölümler aslında bir önceki başlıkta gördüğümüz yapıları işaret eder. Ne demek istiyorsun? Aşağıdaki diyagramda demek istenileni görebilirsiniz:

![delay_import_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/delay_import_01.png)

Delay Load sadece fonksiyon ilk kez çağrıldığında gerçekleşir. Ayroca tüm modülün fonksiyonlarını tek seferde değil, sadece ihtiyaç duyulan fonksiyon yüklenir.

Delay Load için yine windbg ile analiz yapmak istersek:

```cpp
lkd> dx -id 0,0,ffffdb095c3f3080 -r1 (*((ntkrnlmp!_IMAGE_DATA_DIRECTORY *)0x7ff69f1b01f0))
(*((ntkrnlmp!_IMAGE_DATA_DIRECTORY *)0x7ff69f1b01f0))                 [Type: _IMAGE_DATA_DIRECTORY]
    [+0x000] VirtualAddress   : 0x2c9d8 [Type: unsigned long]
    [+0x004] Size             : 0xe0 [Type: unsigned long]
lkd> dc /c8 00007ff69f1b0000+2c9d8
00007ff6`9f1dc9d8  00000001 00027380 00030c78 00035000 0002cab8 0002cf48 00000000 00000000  .....s..x....P......H...........
00007ff6`9f1dc9f8  00000001 00027400 00031268 00035098 0002cb50 0002cfe0 00000000 00000000  .....t..h....P..P...............
00007ff6`9f1dca18  00000001 00027410 00031270 000350e8 0002cba0 0002d030 00000000 00000000  .....t..p....P......0...........
00007ff6`9f1dca38  00000001 00027420 00031278 00035100 0002cbb8 0002d048 00000000 00000000  .... t..x....Q......H...........
```

Her satır gecikmeli import edilecek DLL'i temsil eder. Bu satırlar da bildiğiniz üzere _IMAGE_DELAYLOAD_DESCRIPTOR'ın verileridir. İlk satırdan devam edelim. Bu satırdaki veriler aşağıdaki gibi bölümlere ayrılır:

- AllAttributes: 00000001

- DllNameRVA: 00027380

- ModuleHandleRVA: 00030c78

- ImportAddressTableRVA: 00035000

- ImportNameTableRVA: 0002cab8

- BoundImportAddressTableRVA: 0002cf48

- UnloadInformationTableRVA: 00000000

- TimeDateStamp: 00000000

Bu bölümlerde öncelikle DLL'isimlerinin RVA'ları takip edelim. Daha sonra ilgili DLL'de delay importlanabilecek fonksiyonları listeleyelim:

```cpp
lkd> da 00007ff69f1b0000+00027380
00007ff6`9f1d7380  "ADVAPI32.dll"
lkd> dps 00007ff69f1b0000+0002cab8
00007ff6`9f1dcab8  00000000`0002cc30
00007ff6`9f1dcac0  00000000`0002cd62
00007ff6`9f1dcac8  00000000`0002cc44
00007ff6`9f1dcad0  00000000`0002cc5a
00007ff6`9f1dcad8  00000000`0002cc78
00007ff6`9f1dcae0  00000000`0002cc8a
00007ff6`9f1dcae8  00000000`0002cc9e
00007ff6`9f1dcaf0  00000000`0002ccae
00007ff6`9f1dcaf8  00000000`0002ccbc
00007ff6`9f1dcb00  00000000`0002cccc
00007ff6`9f1dcb08  00000000`0002ccde
00007ff6`9f1dcb10  00000000`0002ccf0
00007ff6`9f1dcb18  00000000`0002cd04
00007ff6`9f1dcb20  00000000`0002cd14
00007ff6`9f1dcb28  00000000`0002cd2a
00007ff6`9f1dcb30  00000000`0002cd3a
```

Görüldüğü gibi delay importlanabilecek fonksiyonların RVA'ları çekilmiştir. İlk fonksiyonun RVA'sını takip edersek:

```cpp
lkd> dc 00007ff69f1b0000+00000000`0002cc30
00007ff6`9f1dcc30  704f0218 72506e65 7365636f 6b6f5473 6e65  ..OpenProcessToken"
```

Burada aslında _IMAGE_IMPORT_BY_NAME yapısını görüyorsunuz.

- (2-byte) Hint: 02 18

- (4-byte) Name: 4F 70 65 6E 50 72 6F 63 65 73 73

Yine dilerseniz XNTSV uygulamasını kullanarak bu bölüme erişebilirsiniz:

![delay_import_analysis_01.PNG](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/delay_import_analysis_01.png)

# PE: Export

Import başlığından sonra bu başlık size daha kolay gelecek, merak etmeyin :) Import başlığında anlatılanları anladıysanız, dışarıda varolan bir fonksiyonlar kümesini kendi programımıza dahil ettiğimizi anlamışsınızdır. Export ise bu işin tersi diyebiliriz. Yani dışarı kaynaklardan fonksiyon almak yerine fonksiyon veriyoruz, yani export'luyoruz.

![pe_export_mechanism_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_export_mechanism_01.png)

İlk olarak bakacağımız yapı PE yapısındaki data directory'de bulunan export directory altındaki işaret edilen yapıdır, yani aşağıdaki yapı:

```cpp
typedef struct _IMAGE_EXPORT_DIRECTORY {
    DWORD    Characteristics;
    DWORD    TimeDateStamp;
    WORD    MajorVersion;
    WORD    MinorVersion;
    DWORD    Name;
    DWORD    Base;
    DWORD    NumberOfFunctions;
    DWORD    NumberOfNames;
    DWORD    AddressOfFunctions;
    DWORD    AddressOfNames;
    DWORD    AddressOfNameOrdinals;
} IMAGE_EXPORT_DIRECTORY,*PIMAGE_EXPORT_DIRECTORY;
```

**Characteristics:** Genelde kullanılmaz. 0 değerini gösterir.

**TimeDateStamp:** Export tablosunun oluşturulma zamanını tutar.

**MajorVersion:** Genelde kullanılmaz. 0 değerini tutar.

**MinorVersion:** Genelde kullanılmaz. 0 değerini tutar.

**Name:** Ascii kodlamada export edilen DLL'in ismini barındıran alanın RVA'sını tutar.

**Base:** 

**NumberOfFunctions:** Kaç adet fonksiyonun export edildiğini gösteren alandır.

**NumberOfNames:** Genelde NumberOfFunctions bölümünün tuttuğu değerle aynıdır.

**AddressOfFunctions:** Fonksiyon adreslerinin bulunduğu array'e olan RVA'sını tutar. Export Address Table olarakta geçer.

**AddressOfNames:** Fonksiyon isimlerinin bulunduğu array'e olan RVA'sını tutar. Export Name Pointer Table olarakta geçer.

**AddressOfNameOrdinals:** Sıra numaralarının bulunduğu array'e olan RVA'sını tutar. Export Ordinal Table

Normalde PE dosyalarının Export Directory'leri boş olduğundan, bizim import ettiğimiz (yani import edilen dosyadaki export edilenler) modüllere yoğunlaşmamız gerekmektedir. Biliyorsunuz ki DLL dosyaları da PE yapısındadır. Diskteki bir DLL'i export directory kısmına erişmek kolay (herhangi bir parser'a ilgili DLL'i sürükleyin) o yüzden bunu atlayacağım. Bellekteki bir DLL'i analiz etmek için aşağıdaki yöntemleri kullanabilirsiniz:

- Belleğe map eden bir PE dosyası için önce kullandığı modüllere XNTSV ile bakalım.

![export_memory_analysis_pe_modules_01.PNG](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/export_memory_analysis_pe_modules_01.png)

- Herhangi bir debugger ile process'e attach edip moduller kısmından bulunabilir.

![export_memory_analysis_pe_modules_02.PNG](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/export_memory_analysis_pe_modules_02.png)

İstenilen modüllerden birisinin adresi kopyalanıp windbg ile analiz edilebilir. Bu yazıda export tablosu incelenirken KERNEL32.dll ele alınmıştır.

Öncelikle ilgili DLL'in image base'inden export directory'e erişelim ve export data directory yapısını tutan RVA'yı bulalım:

```cpp
lkd> dc /c A 7ffaec500000+9a330
00007ffa`ec59a330  00000000 6024807a 00000000 0009e322 00000001 00000661 00000661 0009a358 0009bcdc 0009d660  ....z.$`....".......a...a...X.......`...
```

Konunun başında anlatılan bölümleri bu değerlerle eşleştirmek istersek:

- (4-byte) Characteristics: 00000000

- (4-byte) TimeDateStamp: 6024807a

- (2-byte) MajorVersion: 0000

- (2-byte) MinorVersion: 0000

- (4-byte) Name: 0009e322

- (4-byte) Base: 00000001

- (4-byte) NumberOfFunctions: 00000661

- (4-byte) NumberOfNames: 00000661

- (4-byte) AddressOfFunctions: 0009a358

- (4-byte) AddressOfNames: 0009bcdc

- (4-byte) AddressOfNameOrdinals: 0009d660

İlk olarak aşağıdaki bölümden başlamak istedim. Çünkü burası export edilen fonksiyonu belirleyen asıl kısımdır, yani sıra numarası. Bu numaralar mantıken en fazla NumberOfFunctions/NumberOfNames bölümünde tutulan değerle aynıdır (0'dan başladığı için bir eksiği olacaktır).

![export_structure_memory_analysis_ordinals_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/export_structure_memory_analysis_ordinals_01.png)

Bu kısımdaki numaralar export edilen fonksiyonların sıra numarası olduğunu söylemiştik. O halde örnek olması açısından 8. sıra numarasını tutalım. Ordinal dizisinde 8 kere ilerlediğimizde 7. index'i elde ediyoruz, kafanız karışmasın.

Daha sonra fonksiyon isimlerinin tutulduğu array'i alalım ve 7. index'te yer alan RVA'ya gidelim:

```cpp
lkd> dc 7ffaec500000+0009bcdc
00007ffa`ec59bcdc  0009e32f 0009e368 0009e39b 0009e3aa  /...h...........
00007ffa`ec59bcec  0009e3bf 0009e3c8 0009e3d1 0009e3e2  ................
00007ffa`ec59bcfc  0009e3f3 0009e438 0009e45e 0009e47d  ....8...^...}...
00007ffa`ec59bd0c  0009e49c 0009e4a9 0009e4bc 0009e4d4  ................
00007ffa`ec59bd1c  0009e4ef 0009e504 0009e521 0009e560  ........!...`...
00007ffa`ec59bd2c  0009e5a1 0009e5b4 0009e5c1 0009e5db  ................
00007ffa`ec59bd3c  0009e5f9 0009e630 0009e675 0009e6c0  ....0...u.......
00007ffa`ec59bd4c  0009e71b 0009e770 0009e7c3 0009e818  ....p...........
lkd> da 00007FFAEC500000+0009e3e2
00007ffa`ec59e3e2  "AddConsoleAliasW"
```

Görüldüğü gibi 7. index'teki fonksiyonun ismini AddConsoleAlias olarak getirdi. Şimdi ise bu fonksiyonun adresini, fonksiyon adreslerinin rva dizisinde yine sıra numarası sayesinde (7. index) bulalım:

```cpp
lkd> dc 7ffaec500000+0009a358
00007ffa`ec59a358  0009e347 0009e37d 000207e0 0001be70  G...}.......p...
00007ffa`ec59a368  0005a6f0 000128f0 00025da0 00025db0  .....(...]...]..
00007ffa`ec59a378  0009e403 0003d290 0005a830 0005a890  ........0.......
00007ffa`ec59a388  000229d0 0001ea20 0003abd0 00021000  .).. ...........
00007ffa`ec59a398  0003abf0 00038ff0 0009e53c 0009e57c  ........<...|...
00007ffa`ec59a3a8  00007200 000259f0 0003ac30 0003ac10  .r...Y..0.......
00007ffa`ec59a3b8  0009e60f 0009e64d 0009e695 0009e6e8  ....M...........
00007ffa`ec59a3c8  0009e740 0009e794 0009e7e8 0009e833  @...........3...
lkd> dps 00007FFAEC500000+00025db0
00007ffa`ec525db0  cccc0005`bfea25ff
00007ffa`ec525db8  cccccccc`cccccccc
```

Bunları daha kolay görebilmek için x32/64dbg kullanabilirsiniz:

![export_memory_analysis_pe_modules_03.PNG](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/export_memory_analysis_pe_modules_03.png)

# ~~PE: Relocation~~

# PE: Resource

PE dosyasının içerisinde göümülü olarak gelen verileri temsil etmek için böyle bir yapı çıkarılmış. Örneğin bir program indirdiğinizde icon dosyasını program dosyaları arasında göremeyebilirsiniz, ama program bir icon'a sahip. Nasıl? Veya program metadata gibi bilgileri, gömülü (embeded) resimler, diyaloglar vs. hepsi pe yapısında resource'ta saklanır.

Resource bölümünü 5 ana bölüme ayırmak istersek:

- Root: PE Resource bölümünün başlangıç bölümüdür. Bu kısım _IMAGE_RESOURCE_DIRECTORY yapısı ile temsil edilir.

- Level 1 (Type): Bu kısım resource'un tipini belirten kısımdır. Root'un hemen sonrasında _IMAGE_RESOURCE_DIRECTORY yapısının içerisindeki NumberOfEntry'ler toplamı kadar _IMAGE_RESOURCE_DIRECTORY_ENTRY yapısı bellekte ardı ardına yerleştirilir.
  
  > Internal'de resource tiplerini belirten flag'ler vardır. Bunları görebilmek için: [LIEF - Resource Types (Winuser.h)](https://lief-project.github.io/doc/stable/api/python/pe.html#lief.PE.RESOURCE_TYPES)

- Level 2 (ID/String): 

- Level 3 (Country/Language): 

- Raw Data

Bu yapıların birbirlerine bağlı bir ağaç modeli olarak düşünebilirsiniz.

![pe_resource_mechanism_01.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_resource_mechanism_01.png)

Aklınızda daha iyi oturması açısından bir şey eklemek istiyorum; resource bölümü aslında iki farklı structure ile ifade edilebilir. Bunlar:

- Resource Directory: Yani bizim ağaç modelimizde dallanabilen leaf adı verilen (level 1, 2, 3) entry'ler. Ağaç modelinde gördüğünüz Manifest, Icon, ID 1, MY_NAME, Language gibi gibi.

- Resource Data: Bu bölüm ise ağacımızdaki yaprak dediğimiz leaf denilen son datalardır yani raw data, ham data'lardır, asıl ulaşılmak istenen data'lardır.

> Ek bir şey daha; ağaç diyagramındaki bir dalın iki farklı dala gittiğini görmüşsünüzdür (üç veya daha fazla dala da ilerleyebilirdi). Bu şimdilik kafanızı karıştırmasın. Yazının ilerisinde bu konuya değinilmiştir.

Şimdi de bu diyagramı aşağıdaki resource bölümünün oluşturduğu yapılara uyarlayalım:

```cpp
typedef struct _IMAGE_RESOURCE_DIRECTORY {
    DWORD    Characteristics;
    DWORD    TimeDateStamp;
    WORD    MajorVersion;
    WORD    MinorVersion;
    WORD    NumberOfNamedEntries;
    WORD    NumberOfIdEntries;
    /*  IMAGE_RESOURCE_DIRECTORY_ENTRY DirectoryEntries[]; */
} IMAGE_RESOURCE_DIRECTORY,*PIMAGE_RESOURCE_DIRECTORY;
```

Bu yapı pe yapısının resource bölümündeki root yapısı olduğunu söylemiştik. Yani resource section header altındaki RVA bu yapıyı işaret eder. Buradaki bölümlerden:

**NumberOfNamedEntries:** İsimlendirilmiş girdilere sahip dallanma sayısıdır.

**NumberOfIdEntries:** Tamsayı ile ifade edilen girdilere sahip dallanma sayısıdır.

Yani bu iki bölümün toplamı bizim pe yapısındaki resource bölümünün toplam directory entry sayısını söyler.

Aşağıdaki yapı ise bizim sabahtandır bahsettiğimiz directory yani dallanabilen (node) yapısıdır.

```cpp
#define    IMAGE_RESOURCE_NAME_IS_STRING        0x80000000
#define    IMAGE_RESOURCE_DATA_IS_DIRECTORY    0x80000000

typedef struct _IMAGE_RESOURCE_DIRECTORY_ENTRY {
    union {
        struct {
            unsigned NameOffset:31;
            unsigned NameIsString:1;
        } DUMMYSTRUCTNAME;
        DWORD   Name;
        WORD    Id;
    } DUMMYUNIONNAME;
    union {
        DWORD   OffsetToData;
        struct {
            unsigned OffsetToDirectory:31;
            unsigned DataIsDirectory:1;
        } DUMMYSTRUCTNAME2;
    } DUMMYUNIONNAME2;
} IMAGE_RESOURCE_DIRECTORY_ENTRY,*PIMAGE_RESOURCE_DIRECTORY_ENTRY;
```

Bu yapı Level 1'de de var Level 2'de de 3'te de vardır.

```cpp
typedef struct _IMAGE_RESOURCE_DATA_ENTRY {
    DWORD    OffsetToData;
    DWORD    Size;
    DWORD    CodePage;
    DWORD    Reserved;
} IMAGE_RESOURCE_DATA_ENTRY,*PIMAGE_RESOURCE_DATA_ENTRY;
```

Her başlık altında windbg ile örnekler gösterdiğimi biliyorum, belki bundan sıkılmış olabilirsiniz ama PE konusu hakkında okuduğum yazıların neredeyse hiçbirisi windbg ile örnekler yapmamış. Okuyucuya da farklılık olması için memory'deki bir imajı byte-byte analiz etmenin pratiklik kazandıracağına inanıyorum. Herkes hazıra alışır da parser'lara kalırsa bu işin zevki nerede kalır... Lafı çok uzattım, hemen windbg ile analize başlayalım :)

İlk olarak Resource Data Directory kısmındaki RVA değerimizi alalım. Bu değer bildiğiniz üzere .rsrc section'ın başlangıç adresini elde etmemizi sağlayacak. (Bu arada section'ın ismi farklılık gösterebilir .rsrc dediğime bakmayın, genelde öyle denir işte)

```cpp
kd> dx -id 0,0,ffffc703ac9ef080 -r1 (*((ntkrnlmp!_IMAGE_DATA_DIRECTORY *)0x7ff76d0e0198))
(*((ntkrnlmp!_IMAGE_DATA_DIRECTORY *)0x7ff76d0e0198))                 [Type: _IMAGE_DATA_DIRECTORY]
    [+0x000] VirtualAddress   : 0x36000 [Type: unsigned long]
    [+0x004] Size             : 0xbd8 [Type: unsigned long]
```

Bu kısımdan sonra dikkatli okumanızı istiyorum çünkü işler biraz karışacak.

```cpp
lkd> dc /c 4 00007ff76d0e0000+36000
00007ff7`6d116000  00000000 00000000 00000000 00020003  ................
...
...
```

Burası bizim karşımıza ilk çıkan Root dediğimiz kısım. Değerleri _IMAGE_RESOURCE_DIRECTORY yapısına uyarladığımızda:

- (4-byte) Characteristics: 00000000

- (4-byte) TimeDateStamp: 00000000

- (2-byte) MajorVersion: 0000

- (2-byte) MinorVersion: 0000

- (2-byte) NumberOfNamedEntries: 0003

- (2-byte) NumberOfIdEntries: 0002

Hemen ne düşündüğünüzü tahmin eder gibiyim :) Evet, 5 tane directory entry'e sahibiz. Bunu kendimizce sağlama almak adına aşağıdaki gibi image'ın resource bölümü görüntülenmiştir:

![pe_resource_analyzing_03.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_resource_analyzing_03.png)

CFF aracı bu section'ı anlatmak/anlamak açısından çok iyidir. İsteyenler daha basit görünümdeki RHacker'da kullanabilir.

> Bu arada bir image'ın diskteki resource'u ile memory'deki resource'u farklılık gösterebilir, veya resource'ta olmaması gereken dosyalar olabilir. 

Devam edelim... windbg ile her entry'i incelemek uzun süreceğinden sadece ilk entry'den gideceğim.

İlk entry'in Level 1 kısmına baktığımızda bu dataların _IMAGE_RESOURCE_DIRECTORY_ENTRY yapısına ait olduğunu biliyorsunuzdur.

```cpp
lkd> dc /c 4 00007ff76d0e0000+36000+0n16
00007ff7`6d116010  80000178 80000038 800001e6 80000050  x...8.......P...
00007ff7`6d116020  80000250 80000068 00000010 80000080  P...h...........
00007ff7`6d116030  00000018 80000098 00000000 00000000  ................
00007ff7`6d116040  00000000 00000001 800001a8 800000b0  ................
00007ff7`6d116050  00000000 00000000 00000000 00000001  ................
00007ff7`6d116060  80000214 800000c8 00000000 00000000  ................
00007ff7`6d116070  00000000 00010000 00000001 800000e0  ................
00007ff7`6d116080  00000000 00000000 00000000 00010000  ................
```

- (4-byte) Name/Id: 80000178

- (4-byte) OffsetToData: 80000038

Şimdi gelin level 1 datalarını takip edelim:

```cpp
lkd> dc 00007ff76d0e0000+36000+0178
00007ff7`6d116178  00450017 00500044 004e0045 0049004c  ..E.D.P.E.N.L.I.
00007ff7`6d116188  00480047 00450054 0045004e 00410044  G.H.T.E.N.E.D.A.
00007ff7`6d116198  00500050 004e0049 004f0046 00440049  P.P.I.N.F.O.I.D.
```

İlk byte'lara dikkat ederseniz önce 17 gibi bir hex sayı, daha sonra entry dir ismi geliyor. Peki nedir bu 17? Tam olarak bu:

```cpp
typedef struct _IMAGE_RESOURCE_DIRECTORY_STRING {
    WORD    Length;
    CHAR    NameString[ 1 ];
} IMAGE_RESOURCE_DIRECTORY_STRING,*PIMAGE_RESOURCE_DIRECTORY_STRING;

typedef struct _IMAGE_RESOURCE_DIR_STRING_U {
    WORD    Length;
    WCHAR    NameString[ 1 ];
} IMAGE_RESOURCE_DIR_STRING_U,*PIMAGE_RESOURCE_DIR_STRING_U;
```

Structure isminden de anlaşılacağı üzere bir directory'in ismini belirtir ve öncesinde isim length'ini alır, bizim 17 de ordan geliyor. 

> Ayrıca iki tane olması kafanızı karıştırmasın sadece ascii kodlama dışında unicode desteği U prefix'i ile gösterilir. Yani olurda directory ismi ascii'den başka bir şey oldu, U prefix'i olan eleman devreye girer.

Level 1 için ismi EDPENLIGHTENEDAPPINFOID olarak bulduk, peki Level 2'ye nasıl geçicez? Başta bulduğumuz OffsetToData değeri sayesinde. Bu değeri takip ettiğinizde yeni bir _IMAGE_RESOURCE_DIRECTORY_ENTRY yapısıyla karşılaşacaksınız:

```cpp
lkd> dc 00007ff76d0e0000+36000+0038
00007ff7`6d116038  00000000 00000000 00000000 00000001  ................
```

Şimdi burada yine değerleri yerleştirmek istemiyorum, direkt tipinden de anlaşılacağı üzere bir adet named directory mevcut diyor.

> Bu arada konunun başında ağaç diyagramı için bir daldan 2 adet dallanma (3 ve fazlası da olabilirdi) olduğundan kafanız karışmasın demiştim. Burası o kısım işte.

Level 2 direkt olarak gösterilmek istenirse:

```cpp
lkd> dc 00007ff76d0e0000+36000+0038+10
00007ff7`6d116048  800001a8 800000b0 00000000 00000000  ................
lkd> dc 00007ff76d0e0000+36000+01a8
00007ff7`6d1161a8  004d001e 00430049 004f0052 004f0053  ..M.I.C.R.O.S.O.
00007ff7`6d1161b8  00540046 00440045 00450050 004c004e  F.T.E.D.P.E.N.L.
00007ff7`6d1161c8  00470049 00540048 004e0045 00440045  I.G.H.T.E.N.E.D.
00007ff7`6d1161d8  00500041 00490050 0046004e 0016004f  A.P.P.I.N.F.O...
```

Level 3 ise aşağıdaki gibidir:

```cpp
lkd> dc 00007ff76d0e0000+36000+00b0
00007ff7`6d1160b0  00000000 00000000 00000000 00010000  ................
```

Burada ise language leveli olduğu için id verilir, language için name verilir mi bilmiyorum, her neyse. 1 adet Id directory'e sahibiz, hemen bakalım buna da.

```cpp
lkd> dc 00007ff76d0e0000+36000+00b0+10
00007ff7`6d1160c0  00000409 00000128 00000000 00000000  ....(...........
```

Burada da artık noluyorsa tam anlayamadım ama 409 bir offset değil de verinin kendisi olarak gösteriliyor sanırım. Yani decimal karşılığı 1033 oluyor ve bu sanırsam bir language/sub-language bayrağı demek oluyor. Language flaglerine [buradan](https://lief-project.github.io/doc/stable/api/python/pe.html#lief.PE.RESOURCE_LANGS) veya [buradan](https://lief-project.github.io/doc/stable/api/python/pe.html#lief.PE.RESOURCE_SUBLANGS) bakabilirsiniz. Son olarak level 3'ten raw dataya geçmek için _IMAGE_RESOURCE_DATA_ENTRY yapısını ele almamız gerekiyor:

```cpp
lkd> dc /c 1 00007ff76d0e0000+36000+128
00007ff7`6d116128  00036710  .g..
00007ff7`6d11612c  00000002  ....
00007ff7`6d116130  00000000  ....
00007ff7`6d116134  00000000  ....
```

- (4-byte) OffsetToData: 00036710

- (4-byte) Size: 00000002

- (4-byte) CodePage: 00000000

- (4-byte) Reserved: 00000000

İlgili offset'i takip ettiğimizde ise asıl dataya erişmiş oluyoruz:

```cpp
lkd> db 00007ff76d0e0000+00036710
00007ff7`6d116710  01 00 00 00 00 00 00 00-01 00 00 00 00 00 00 00  ................
```

Raw datanın boyutu 2 byte dendiği için offsette çıkan veriden itibaren 2 byte tek okuyacağız. Onca analiz uğraş sadece "01 00" içindi yani, süpermiş. Son olarak analiz ettiğimiz verileri sağlama alalım:

![pe_resource_analyzing_04.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_resource_analyzing_04.png)

![pe_resource_analyzing_05.png](/assets/img/2023-04-09-windows-portable-executable-pe-dosyaları/pe_resource_analyzing_05.png)

# ~~PE: Exception~~

# ~~PE: Debug~~

# ~~PE: Load Config~~

# ~~PE: .NET Metadata~~

```cpp
typedef struct IMAGE_COR20_HEADER
{
    DWORD cb;
    WORD  MajorRuntimeVersion;
    WORD  MinorRuntimeVersion;

    IMAGE_DATA_DIRECTORY MetaData;
    DWORD Flags;
    union {
        DWORD EntryPointToken;
        DWORD EntryPointRVA;
    } DUMMYUNIONNAME;

    IMAGE_DATA_DIRECTORY Resources;
    IMAGE_DATA_DIRECTORY StrongNameSignature;
    IMAGE_DATA_DIRECTORY CodeManagerTable;
    IMAGE_DATA_DIRECTORY VTableFixups;
    IMAGE_DATA_DIRECTORY ExportAddressTableJumps;
    IMAGE_DATA_DIRECTORY ManagedNativeHeader;

} IMAGE_COR20_HEADER, *PIMAGE_COR20_HEADER;
```

```cpp
typedef enum ReplacesCorHdrNumericDefines
{
    COMIMAGE_FLAGS_ILONLY           = 0x00000001,
    COMIMAGE_FLAGS_32BITREQUIRED    = 0x00000002,
    COMIMAGE_FLAGS_IL_LIBRARY       = 0x00000004,
    COMIMAGE_FLAGS_STRONGNAMESIGNED = 0x00000008,
    COMIMAGE_FLAGS_NATIVE_ENTRYPOINT= 0x00000010,
    COMIMAGE_FLAGS_TRACKDEBUGDATA   = 0x00010000,
    COMIMAGE_FLAGS_32BITPREFERRED   = 0x00020000,

    COR_VERSION_MAJOR_V2       = 2,
    COR_VERSION_MAJOR          = COR_VERSION_MAJOR_V2,
    COR_VERSION_MINOR          = 5,
    COR_DELETED_NAME_LENGTH    = 8,
    COR_VTABLEGAP_NAME_LENGTH  = 8,

    NATIVE_TYPE_MAX_CB = 1,
    COR_ILMETHOD_SECT_SMALL_MAX_DATASIZE = 0xff,

    IMAGE_COR_MIH_METHODRVA  = 0x01,
    IMAGE_COR_MIH_EHRVA      = 0x02,
    IMAGE_COR_MIH_BASICBLOCK = 0x08,

    COR_VTABLE_32BIT             = 0x01,
    COR_VTABLE_64BIT             = 0x02,
    COR_VTABLE_FROM_UNMANAGED    = 0x04,
    COR_VTABLE_CALL_MOST_DERIVED = 0x10,

    IMAGE_COR_EATJ_THUNK_SIZE = 32,

    MAX_CLASS_NAME   = 1024,
    MAX_PACKAGE_NAME = 1024,
} ReplacesCorHdrNumericDefines;
```

# Kullanılabilir Kaynaklar

PE yapısını akılda kalıcı bir şekilde tanımak için aşağıdaki listeye bakabilirsiniz: [0x0ryctes \| Resources](http://alipasaturhan.github.io/resources/#pe-structure)

# Kaynakça ve Referanslar

1. https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#special-sections

2. [00. PE Import Table Parser](https://stofu.io/blog/view_post.php?id=10)

3. [Exciting Journey Towards Import Address Table (IAT) of an Executable](https://tech-zealots.com/malware-analysis/journey-towards-import-address-table-of-an-executable-file/)

4. [Import Address Tables and Export Address Tables \| Machines Can Think](https://bsodtutorials.wordpress.com/2014/03/02/import-address-tables-and-export-address-tables/)

5. [wine/winnt.h at master · wine-mirror/wine · GitHub](https://github.com/wine-mirror/wine/blob/master/include/winnt.h)

6. [The .NET File Format](https://www.ntcore.com/files/dotnetformat.htm)

7. [Anatomy of a .NET Assembly - CLR metadata 1 - Simple Talk](https://www.red-gate.com/simple-talk/blogs/anatomy-of-a-net-assembly-clr-metadata-1/)

8. [dotnetfile Open Source Python Library: Parsing .NET PE Files Has Never Been Easier](https://unit42.paloaltonetworks.com/dotnetfile/)

9. [Welcome - dotnetfile API documenation](https://pan-unit42.github.io/dotnetfile/)

10. [Resource Types (Winuser.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/menurc/resource-types)
