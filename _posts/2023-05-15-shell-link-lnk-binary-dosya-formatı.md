---
layout: post
title: Shell Link (.lnk) Binary Dosya Formatı
date: 2023-05-15 03:17 +0300
categories: [Dosya Yapıları, Shell Link (.lnk)]
tags: [windows, shell-link-lnk, file-structure]
---

Hepimiz eskiden bir bilgisayardaki programı/oyunu kopyalarken kısayol dosyasını kopyalamak gibi bir hata yapmışızdır :) Şimdi gelin bu kısayol dosyalarından hıncımızı alalım.

LNK dosyaları bilgisayarımızda farklı veya aynı dizinde bulunan yazılımı açmaya yarayan dosyalardır. Bu dosyaları kötüye kullananlar da var normal kullananlarda, kullanmayı bilmeyenler de (bu biz oluyoruz). Bu dosya doğal olarak napar; asıl çalıştırmak istediği programın dizinini (yani hedef/target dosyayı) referans alır ve çağırım işlemini bizim yerimize kendisi yapar (aslında parametre vs. verme olayları var ama ona yazının ilerisinde değineceğiz)

Shell link binary dosya formatı aşağıda gösterildiği gibi bir dizi yapıdan/header'dan oluşur:

- SHELL_LINK_HEADER

- [LINKTARGET_IDLIST]

- [LINKINFO]

- [STRING_DATA]

- *EXTRA_DATA

Bu 5'li yazıda tek tek ele alınmıştır (gerekli kısımları tabii :)).

# ShellLinkHeader

.lnk'nın ilk header'ı olmazsa olmazımız (gerçekten olmazsa olmaz bu arada, yani optional bir başlık değil) dosya hakkında kritik bilgiler barındırıyor. Dosyanın oluşum tarihi, editlenme tarihi, ilk açıldığında tam ekran mı açılacak, kısayol desteği varmı vb. Bunlar size şimdilik kritik bilgiden çok gereksiz gibi geliyorsa biraz da zararlı lnk dosya analizlerini araştırın derim.

> Bu header'ın optional olmamasının güzel bir yanı da parser yazmak isteyenler için ekstra if-else koşullarıyla uğraşmaya gerek yok ve zaten lnk'nın ilk/baş yapısı olduğu için yapı içerisindeki bölümlerin tiplerine göre ilerleyerek bodoslama parse edilebilir :) Diğer header'lar optional olduğu için belirli flag'ler set edilmiş mi diye kontrol ettirmemiz gerekiyor, maalesef. 

Aşağıda bu header'ın C++ yapısı verilmiştir. Her ne kadar msdn bu yapıyı bizlerden gizlemeye çalışsa da, çareyi didier stevens abimizin hazırladığı lnk parser kaynak kodunda buldum (sanırım yazı boyunca da böyle yapıcam).

```cpp
#define LINK_SIZEOF_CLSID 16

typedef struct
{
    DWORD HeaderSize;                     /* Must be 0x0000004C */
    BYTE LinkCLSID[LINK_SIZEOF_CLSID];    /* This value MUST be 00021401-0000-0000-C000-000000000046. */
    DWORD LinkFlags;
    DWORD FileAttributes;
    ULONG CreationTime;
    ULONG AccessTime;
    ULONG WriteTime;
    DWORD FileSize;
    DWORD IconIndex;
    enum ShowCommand {
            SW_SHOWNORMAL = 0x00000001,
            SW_SHOWMAXIMIZED = 0x00000003,
            SW_SHOWMINNOACTIVE = 0x00000007
    };
    WORD HotKey;
    WORD Reserved;
    DWORD Reserved;
    DWORD Reserved;
} ShellLinkHeader;
```

Bu yapının bölümlerini madde madde açıklamak istersek:

**HeaderSize:** Ait olduğu yapının boyutudur. Değerinin 4C olmak zorunluluğu sizce de garip değil mi?

**LinkCLSID:** Dosyanın kimlik numarası gibi düşünülebilir. Değeri 00021401-0000-0000-C000-000000000046 olmak zorundadır.

**LinkFlags:** Lnk dosyamızın optional header'larından/özelliklerinden hangileri kullanılmış sorusunun cevabını bit maskeleme yöntemiyle açıklar. Yani arkada bir sürü flag olduğunu düşünün ve her birinin kendine ait değeri olduğunu, bu değerleri tek tek bir for döngüsüyle LinkFlags içerisindeki değerle AND işlemine sokarsanız, hangi flag'lerin açık olduğunu görebilirsiniz, bit maskeleme işte bu.

> Tüm flag'leri görebilmek için: [SHELL_LINK_DATA_FLAGS (shlobj_core.h)](https://learn.microsoft.com/en-us/windows/win32/api/shlobj_core/ne-shlobj_core-shell_link_data_flags) veya [[MS-SHLLINK]: LinkFlags](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/ae350202-3ba9-4790-9e9e-98935f4ee5af?redirectedfrom=MSDN)

**FileAttributes:** İşaret edilen dosyanın özniteliğini tutar. Mesela dosya şifreli mi? geçici bir dosya mı? Bir klasör mü? gibi gibi. Bu da LinkFlags gibi bit maskeleme yöntemiyle kendine gelebilecek flagleri ifade eder.

> Tüm flag'leri görebilmek için: [File Attribute Constants (WinNT.h)](https://learn.microsoft.com/en-us/windows/win32/fileio/file-attribute-constants) veya [[MS-SHLLINK]: FileAttributesFlags](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/378f485c-0be9-47a4-a261-7df467c3c9c6?redirectedfrom=MSDN)

**CreationTime:** UTC olarak lnk'nın oluşturulma tarihini/zamanını tutar. 

**AccessTime:** UTC olarak lnk'nın son erişim tarihini/zamanını tutar.

**WriteTime:** UTC olarak lnk'nın son editlenme tarihini/zamanını tutar.

**FileSize:** Hedef açılacak programın boyutunu tutar. Ama u32 bit olduğu için bazı karışıklıklar çıkabilir. Eğer açılacak program 0xF0xFFFFFFFF'den (4.294.967.295) büyükse, buradaki değer hedef dosya boyutunun en az anlamlı (LSB) 32 bitini belirtir.

**IconIndex:** Bu kısım bana eğlenceli geliyor nedendir bilmiyorum.. IconIndex içerisindeki değer aslında İlgili .lnk dosyasına r_click > properties ve shortcut sekmesinden change_icon dediğinizde kaçıncı iconun seçili olduğunu belirtir. Ben kendimce bu kısmı ikiye ayırmak istiyorum:

- Hedef uygulama, simgesini external olarak .ico dosyasından çekiyorsa
  
  - O zaman mantıken 1 tane icon dosyası olacağından IconIndex 0 olmak zorundadır (belki istisnalar vardır bilemicem)

- Hedef uygulama, simgesini internal olarak resource'undan çekiyorsa
  
  - Birden fazla icon dosyasının olma ihtimali vardır. Bu da demek oluyor ki properties bölümünden change_icon dediğimizde birden fazla icon'un olduğunu görebilme ihtimalimiz vardır.

Bu kısmı detaylı açıklamak gereksiz ve uzun olur, o yüzden isteyenler RHacker aracını kullanarak PE resource'u inceleyebilir veya resource nedir? diye soran olursa önceki paylaştığım [windows pe yapısı](/posts/windows-portable-executable-pe-dosyaları/#pe-resource) yazıma bakabilir.

**ShowCommand:** Lnk tarafından başlatılan uygulamanın pencere durumunu belirtir. Buraya zaten mantıken 3 tane değer gelebilir (Normal, Maximized, Minimized)

**HotKey:** Lnk dosyasındaki bu bölümde belirtilen kısayol tuşları sayesinde hedef programı zahmetsizce açabilirsiniz (o kadar da tembel değiliz microsoft :))

> Tüm hotkey flag'lere ulaşmak için: [[MS-SHLLINK]: HotKeyFlags](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink/8cd21240-1b5d-43e6-adc4-38cf14e30cea?redirectedfrom=MSDN)

Reserved dediği bölümler yapıya gelebilecek olası güncellemeyle yeni bölümlere yer ayırmak için veya bizden yine bişeyler gizlemek için (belkide sadece diskimizi işgal etmek içindir) kullanılır. Genelde de 0 değerine sahiptirler.

Bu başlığı güzel bir örnekle kapatıyorum, aşağıda herhangi bir lnk dosyasının shell link header kısmını parse edebilen python script'i yazdım:

```python
import binascii

"""
LINK_FLAGS = {}
FILE_ATTRIBUTE_FLAGS = {}
SHOW_COMMAND_VALUES = {}
HOTKEY_FLAGS = {}
"""

SHELL_LINK_HEADER = {"HEADER_SIZE":[0x0, 0x4],
                     "LINK_CLSID":[0x0,0x10],
                     "LINK_FLAGS":[0x0,0x4],
                     "FILE_ATTRIBUTES":[0x0,0x4],
                     "CREATION_TIME":[0x0,0x8],
                     "ACCESS_TIME":[0x0,0x8],
                     "WRITE_TIME":[0x0,0x8],
                     "FILE_SIZE":[0x0,0x4],
                     "ICON_INDEX":[0x0,0x4],
                     "SHOW_COMMAND":[0x0,0x4],
                     "HOT_KEY":[0x0,0x2],
                     "RESERVED_1":[0x0,0x2],    
                     "RESERVED_2":[0x0,0x4],
                     "RESERVED_3":[0x0,0x4]}

def read_bytes(file_path):
    with open(file_path, "rb") as file:
        return file.read()

def parse_bytes(file_bytes):
    cursor = 0
    for i in SHELL_LINK_HEADER:
        SHELL_LINK_HEADER[i][0]=file_bytes[cursor:cursor+SHELL_LINK_HEADER[i][-1]] 
        cursor += SHELL_LINK_HEADER[i][-1]

    return SHELL_LINK_HEADER

def pretty_print(dispersed_data):
    for data in dispersed_data:
        print(data+"["+str(dispersed_data[data][-1])+"]",":",binascii.hexlify(dispersed_data[data][0], " ").decode('ascii').upper())

pretty_print(parse_bytes(read_bytes(input("File Path> "))))
```

Örnek olarak masaüstümde bulduğum rastgele bir lnk dosyasını analiz edelim:

![shell_link_header_parser_output_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/shell_link_header_parser_output_01.png)

> Little endian'dan dolayı byte'lar tersten yazılıyor, kafanız karışmasın.

# ~~LinkTargetIDList~~

# ~~LinkInfo~~

# StringData

String data denince aklınıza hemen "çok şükür byte'lardan kurtulduk" gibi bir şey gelmesin çünkü bu bölüm yine kendi içinde farklı bölümlere dallanıyor ve bu bölümler de önceki bahsettiğimiz LinkFlags'lere bağlı olarak kullanılır, yani aşağıdaki her string data bölümü optional'dır:

- NameString

- RelativePath

- WorkingDir

- CommandLineArguments

- IconLocation

Bu bölümlerin sahip olduğu ortak bir structure vardır, o da şuna benziyor:

```cpp
typedef struct
{
    WORD CountCharacters;
    WCHAR String[CountCharacters];
} StringData;
```

Aşağıda StringData bölümleri açıklanmıştır ayrıca bu bölümleri aktif eden LinkFlags değerleri de yazılmıştır

![string_data_headers_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/string_data_headers_01.png)

Yani StringData diye bir yapı yok aslında, sadece bu yapının bölümleri gerçektir. HasStringData gibi bir ibare göremezsiniz. Şimdi bu bölümleri tek tek ele alalım bakalım:

![lnk_file_property_fields_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/lnk_file_property_fields_01.png)

Bu bölümler ilgili flag'i ile beraber aşağıda tek tek ele alınmıştır.

## NAME_STRING (HasName)

Flag olarak HasName'e bağlıdır. LNK dosyasının comment'idir. Windows sistemlerinizde herhangi bir lnk dosyasının özelliklerine baktığınızda Comment kısmındaki yazıyı içerisinde barındırır (isterseniz kendinize/birilerine not bırakabilirsiniz). Kullanıcılar istediği gibi bu kısmı değiştirebilir.

## RELATIVE_PATH (HasRelativePath)

LNK dosyasının hedefindeki dosyanın path'ini string tipinde saklar. HasRelativePath bayrağına bağlıdır. Yine LNK özelliklerden bu kısmı değiştirebilirsiniz.

## WORKING_DIR (HasWorkingDir)

LNK dosyasının başlangıç yeri/çalışma yeri adresini tutar. LNK özelliklerden Start in kısmından kullanıcılar istedikleri gibi değiştirebilirler. Bu yapı da HasWorkingDir bayrağına bağlıdır.

## COMMAND_LINE_ARGUMENTS (HasArguments)

LNK dosyası çalıştıracağı programa ek parametre/argüman verebilir. Vereceği parametreleri bu bölümde saklar. HasArguments bayrağına bağlıdır. Aşağıda örnek olması açısından ne işe yaradığını bilmediğim bir parametre verdim (sadece parametre olarak algılamasını istediğim için)

Parametre vermeden önceki lnk dosyasının görüntüsü

![string_data_cmd_line_args_before_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/string_data_cmd_line_args_before_01.png)

Parametre verdikten sonraki lnk dosyasının görüntüsü

![lnk_adding_cmd_parameter_string_data_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/lnk_adding_cmd_parameter_string_data_01.png)

![string_data_cmd_line_args_after_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/string_data_cmd_line_args_after_01.png)

Bayrakları kontrol ettiğimizde HasArguments bayrağının 1 set edildiğini görüyoruz.

![link_flags_has_arguments_actived_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/link_flags_has_arguments_actived_01.png)

## ICON_LOCATION (HasIconLocation)

Bu kısım lnk'nın hedefindeki uygulamanın kullandığı icon embeded olup olmamasına bağlıdır. Yani eğer exe'nin resource'una eklenmişse (embeded ise) HasIconLocation 1 set edilir ve hangi exe'den iconun çekildiğinin path'ini tutar. Eğer external olarak .ico dosyası çekiliyorsa bu bölüm kullanılmaz. İsteyenler bunu deneyebilir..

# ExtraData

Bu kısım lnk dosyasının ek datalarının içerdiği header'dır. Yine belirli flag'ler/bitler'in set edilmesiyle devreye giren bölümlere sahiptir ve hepsi tek tek ele alınacaktır. Aslında extra data'nın bölümlerini tek tek ele almayacaktım ama lnk ile ilgili saldırılar artınca analiz yapanlarımız bu bilgilerden eksik olmasın istedim (en azından bildiklerimi aktarmak isterim). Hem DFIR'ciler için önemli bölümlere sahip, hem de kanımca işletim sistemini veya av'leri bu header'ın bölümlerini kullanarak kandırma yöntemlerinin olduğunu düşünüyorum.

Oldukça kapsamlı bir başlık olmasıyla beraber temelde 2 kısımdan oluşur. 

- *EXTRA_DATA_BLOCK

- TERMINAL_BLOCK
  
  - Direkt bu alanı anlatayım ve aradan çıksın istiyorum. Tek vasfı extra data header'ın bitişini belirtmektir. MSDN verilerine göre bu bölümdeki değer unsigned 4 byte ve 0x4'ten küçük olma zorunluluğu varmış (nedendir bilmiyorum, ah msdn ah..)

**EXTRA_DATA_BLOCK[variable]:** Extra Data başlığını oluşturan asıl kısım burasıdır ve 11 bölümden oluşur. Bu başlığı lnk dosyasının bir parçası olarak görmeyiniz (tıpkı StringData'da söylediğimiz gibi). Yani böyle bir extra data block tabloyu byte olarak lnk dosyasının içerisinde göremeyeceksiniz. Sadece 11 adet bölümlerinden set edilenleri görebileceksiniz (byte başlangıcı 11 bölümden birisi olacaktır), her neyse.

> Bu arada `EXTRA_DATA_BLOCK[variable]` kısmında variable denilen eleman aslında extra_data_block'un boyutudur. Extra Data bölümleri de değişkenlik gösterdiğinden boyut farklılığı oluşuyor.

Aşağıda ExtraData Tablosunun bölümleri kısaca açıklanmıştır. Önemli olan başlıklar detaylı bir şekilde üzerinde durulmaya çalışılmıştır.

```cpp
#define EnvironmentVariableDataBlock     0xA0000001
#define ConsoleDataBlock                 0xA0000002
#define TrackerDataBlock                 0xA0000003
#define ConsoleFEDataBlock               0xA0000004
#define SpecialFolderDataBlock           0xA0000005
#define DarwinDataBlock                  0xA0000006
#define IconEnvironmentDataBlock         0xA0000007
#define ShimDataBlock                    0xA0000008
#define PropertyStoreDataBlock           0xA0000009
#define VistaAndAboveIDListDataBlock     0xA000000A
#define KnownFolderDataBlock             0xA000000B
```

Her başlığın yanında kendisine ait flag bitine benzer bir signature değeri vardır.

```cpp
typedef struct tagDATABLOCKHEADER {
  DWORD cbSize;
  DWORD dwSignature;
} DATABLOCK_HEADER, *LPDATABLOCK_HEADER, *LPDBLIST;
```

> Ek bir bilgi vermiş olayım; her bölümün mutlaka BlockSize ve BlockSignature alanları vardır, diğer alanlar ilgili bölüme ait alanlardır.

## EnvironmentVariableDataBlock (0xA0000001)

LNK dosyası için çevresel değişkenler barındırır. Yani tıpkı bir depo gibi düşünebilirsiniz. Lnk'nın çalıştığı an veya sonradan ihtiyacı olabileceği verileri sakladığı alandır.

```cpp
typedef struct {
  DWORD cbSize;
  DWORD dwSignature;
  CHAR  szTarget[MAX_PATH];
  WCHAR swzTarget[MAX_PATH];
} EXP_SZ_LINK, *LPEXP_SZ_LINK;
```

Yukarıdaki yapıda nedense söyleyesim geldi, dikkat etmeniz gereken küçük bir konu var. `char TargetANSI[260]` tam 260 byte'a eşit olurken `wchar_t TargetUnicode[260]` aslında 520 byte'a eşittir. Bunun nedeni sizinde aklınıza gelen daha geniş ve kapsamlı olan unicode kodlamasından kaynaklıdır.

## ConsoleDataBlock (0xA0000002)

Lnk dosyası, konsol penceresinde çalıştırılan bir uygulamayı veya script'i belirttiğinde konsolda kullanılacak görüntü/renk/font (vb.) ayarlarını belirtir. Bu bölümü detaylı incelemek isterseniz

1. Kendinize bir .bat dosyası oluşturun

2. Bu .bat dosyasına ait birtane de lnk dosyası oluşturun.

3. Oluşan lnk dosyasına sağ click yaptığınızda normal bir lnk dosyasında olmayan sekmeler göreceksiniz:
   
   - Terminal
   - Options
   - Font
   - Layout
   - Colors

4. Eğer yukarıdaki sekmelerdeki ayarlar değiştirilmezse, ConsoleDataBlock bölümü normalde lnk dosyasına dahil edilmez. Siz örnek olması için bu sekmelerden herhangi birisindeki değeri değiştirip inceleyebilirsiniz.

```cpp
typedef struct {
  DATABLOCK_HEADER dbh;
  DATABLOCK_HEADER DUMMYSTRUCTNAME;
  WORD             wFillAttribute;
  WORD             wPopupFillAttribute;
  COORD            dwScreenBufferSize;
  COORD            dwWindowSize;
  COORD            dwWindowOrigin;
  DWORD            nFont;
  DWORD            nInputBufferSize;
  COORD            dwFontSize;
  UINT             uFontFamily;
  UINT             uFontWeight;
  WCHAR            FaceName[LF_FACESIZE];
  UINT             uCursorSize;
  BOOL             bFullScreen;
  BOOL             bQuickEdit;
  BOOL             bInsertMode;
  BOOL             bAutoPosition;
  UINT             uHistoryBufferSize;
  UINT             uNumberOfHistoryBuffers;
  BOOL             bHistoryNoDup;
  COLORREF         ColorTable[16];
} NT_CONSOLE_PROPS, *LPNT_CONSOLE_PROPS;
```

COORD yapısı:

```cpp
typedef struct _COORD {
  SHORT X;
  SHORT Y;
} COORD, *PCOORD;
```

COLORREF yapısı:

```cpp
typedef DWORD COLORREF;
typedef DWORD* LPCOLORREF;
```

## TrackerDataBlock (0xA0000003)

Bu alanı kesinlikle bilmekte fayda var. Başta DFIR'ciler için önemli dememin sebebi de bu alandı aslında. Aşağıda bu alanla ilgili bir örnek kodlar görebilirsiniz:

```cpp
typedef struct
{
    DWORD Size;
    DWORD Signature;
    DWORD Length;
    DWORD Version;
    BYTE MachineID[16];
    GUID FileDroid;
    GUID VolumeDroid;
    GUID FileDroidBirth;
    GUID VolumeDroidBirth;
} TrackerDataBlock;
```

**MachineID:** Bu bloğun ilk önemli kısmı, LNK dosyasının en son bulunuduğu makinenin NETBIOS ismini tutar. Nedir bu NETBIOS? diyecek olursanız, kısaca ağda bu makineyi hangi isimle tanıdığımızı belirten yapıdır.

**FileDroid ve FileDroidBirth:** Lnk dosyasının ilk oluştuğu disk'in serial numarasını kapsayan alandır.

**VolumeDroid ve VolumeDroidBirth:** Lnk dosyasının ilk oluştuğu bilgisayarın MAC id'sini tutan alandır. Bu kısmın önemini anlatmama gerek yoktur herhalde.

Kendimce bir senaryo sallıyorum, biz bir tane zararlı lnk yazıp yedirdik ve bizimkini yiyen pc'yi polis karantina altına aldı. Adli bilişim bürosu gelip pc'de ne var ne yok talan ettiklerinde lnk dosyasından dolayı bulaştığımızı anladı ve bu dosya üzerinde yoğunlaştı. Dosyayı ya özel parser'lar kullanır ya extractor'lar bilemem ama zararlı lnk'dan bir adet MAC id ve bir adet Disk Serial id buldu... eveet zaten iş burada bitmiştir. Yarın öbürgün bir yerden patladınız polis bu delilleri kullanabilir. Biz de bu duruma dijital dünyada foot print diyoruz.

Suç taktiklerimizin hemen ardından yazımıza devam ediyoruz :)

## ConsoleFEDataBlock (0xA0000004)

Pek önemli değil gibime geldi ama bildiğim kadarını anlatayım, eğer eski yazılımlar için oluşan lnk dosyalarıyla ilgileniyorsak bu bölüm açılabilir. Çünkü eski yazılımlar unicode kodlaması haricinde özellikle kendisini ilgilendiren bir kodlama sayfasını belirtmeyi isteyebilir. Ama yeni yazılımlar default'ta zaten unicode kodlama tablosunu kullandığından bu bölümün açık olmasına pek rastlamayız (ben hiç rastlamadım, sanırım).

Aşağıda bu bölüme ait structure'ı görebilirsiniz:

```cpp
typedef struct {
  DATABLOCK_HEADER dbh;
  DATABLOCK_HEADER DUMMYSTRUCTNAME;
  UINT             uCodePage;
} NT_FE_CONSOLE_PROPS, *LPNT_FE_CONSOLE_PROPS;
```

Merak edenleriniz olursa kodlama dillerini anlatan official dökümanı [buraya](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-lcid/70feba9f-294e-491e-b6eb-56532684c37f) bırakıyorum.

## SpecialFolderDataBlock (0xA0000005)

Bu başlığı okumadan önce lütfen KnownFolderDataBlock başlığını okuyunuz çünkü ikisi de birbiriyle bağlantılı çalışıyor ve sırasıyla öğrenirsek daha güzel akılda kalıcı olacağını düşünüyorum.

```cpp
typedef struct {
  DWORD cbSize;
  DWORD dwSignature;
  DWORD idSpecialFolder;
  DWORD cbOffset;
} EXP_SPECIAL_FOLDER, *LPEXP_SPECIAL_FOLDER;
```

Bu bölüm KnownFloderDataBlock yapısındaki kimliğin (GUID) CSIDL değerini tutuyor. KnownFolderDataBlock başlığındaki örnekten devam edeceğim. 

1. Aldığımız GUID'i [KnownFolderID Sabitlerinden](https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid#constants) arattık ve `CSIDL Equivalent` sütunundaki değerini bulduk.

2. Daha sonra [CSIDL Tablosundan](https://tarma.com/support/im9/using/symbols/functions/csidls.htm) bu değeri arattığınızda, `idSpecialFolder` içerisindeki değerin tablodan bulduğunuz id olduğunu göreceksiniz.

![special_folder_data_block_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/special_folder_data_block_01.png)

![special_folder_data_block_02.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/special_folder_data_block_02.png)

Veya direkt isteyenler KnownFolderDataBlock'a bakmadan CSIDL tablosunda bu değeri aratarak direkt olarak bulabilirler.

> Bu arada yanlış bilmiyorsam kendi SpecialFolder/KnownFolder'ınıza da oluşturabiliyorsunuz (regedit vs. uğraşarak)

## ~~DarwinDataBlock (0xA0000006)~~

```cpp
typedef struct {
  DATABLOCK_HEADER dbh;
  DATABLOCK_HEADER DUMMYSTRUCTNAME;
  CHAR             szDarwinID[MAX_PATH];
  WCHAR            szwDarwinID[MAX_PATH];
} EXP_DARWIN_LINK, *LPEXP_DARWIN_LINK;
```

## IconEnvironmentDataBlock (0xA0000007)

LNK dosyasının temsil ettiği icon dosyası eğer internal olarak alınıyorsa bu kısım devreye giriyor. Önceki başlıklardan StringData'nın parçası olan Icon Location bölümünü okuduysanız aynı işi yapıyor. Tek farklılığı path'te knwon folder varsa çevirmenlik yapmasıdır.

```cpp
typedef struct
{
    DWORD Size;
    DWORD Signature;
    CHAR TargetANSI[260];
    WCHAR TargetUnicode[260];
} IconEnvironmentDataBlock;
```

Örnek known folder'ların path'leri:

1. %APPDATA%

2. %ProgramData%

3. %ProgramFiles%

Mesela hedef uygulamanın kullandığı ico'ya ulaşmak için `C:\Program Files\Program\program.exe` gibi bir path varsa bunu `%ProgramFiles%\Program\program.exe` haline çeviriyor.

## ~~ShimDataBlock (0xA0000008)~~

```cpp
typedef struct
{
    DWORD Size;
    DWORD Signature;
    WCHAR LayerName[Size-8];
} ShimDataBlock;
```

## PropertyStoreDataBlock (0xA0000009)

Lnk ilk oluştuğu bilgisayar için metdata'lar oluşturur. Bu datalar lnk'nın hedefindeki program veya oluştuğu bilgisayar hakkında olabiliyor. Tahmin edeceğiniz üzere buradaki veriler de digital forensics için kullanılabilir.

```cpp
typedef struct {
  DWORD cbSize;
  DWORD dwSignature;
  BYTE  abPropertyStorage[1];
} EXP_PROPERTYSTORAGE;
```

Burada geçen metadata türleri GUID denilen kimliklerle belirtilir ve kimliklerin de yine id'leri vardır. Bazı metadata türleri şunlardır:

- 46588ae2-4cbc-4338-bbfc-139326986dce -> SID (id: 4)

- 446d16b1-8dad-4870-a748-402ea43d788c -> Volume ID (id: 104)

- b725f130-47ef-101a-a5f1-02608c9eebac -> Date Created (id: 15)

- 28636aa6-953d-11d2-b5d6-00c04fd918d0 -> Parsing Path (id: 30)

LECmd uygulamasını kullanarak çok güzel bir çıktı alabilirsiniz:

![property_store_data_block_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/property_store_data_block_01.png)

Burada geçen metadata türlerini görmek için [libfwps/Windows Property Store](https://github.com/libyal/libfwps/blob/main/documentation/Windows%20Property%20Store%20format.asciidoc) veya [propkey.h](https://github.com/tpn/winsdk-10/blob/master/Include/10.0.14393.0/um/propkey.h) dosyasına bakabilirsiniz.

> Ayrıca SID'nin (user's security identifier) önemi büyük olduğundan nasıl çalıştığını görmekte fayda var: [Security identifiers](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers)

## ~~VistaAndAboveIDListDataBlock (0xA000000A)~~

```cpp
typedef struct
{
    DWORD Size;
    DWORD Signature;
    IDList sIDList;
    WORD TerminalID;
} VistaAndAboveIDListDataBlock;
```

## KnownFolderDataBlock (0xA000000B)

Hani bilgisayarım simgesine çift tıklarsınız ve sol menüde:

- Belgelerim, Müziklerim, Resimlerim

- Çöp Kutusu

- System32

- AppData

- Temp

gibi çok bilinen klasörleriniz gözükür ya (gözükmeyen de vardır bilemicem), bunlara özel klasör deniyor ve microsoft bu özel klasörlerin her birine kimlik (GUID) oluşturup üstüne birer de id atamış, bitti mi? bitmedi tabiki de üstüne bir üstüne sistemsel ad eklemiş ki (Belgelerim => CSIDL_MYPICTURES) biz araştırmacılar daha çok uğraşalım diye...

Aşağıda bu bölüme ait yapıyı görebilirsiniz:

```cpp
typedef struct
{
    DWORD Size;
    DWORD Signature;
    GUID KnownFolderID;
    DWORD Offset;
} KnownFolderDataBlock;
```

Kısa bir örnekle bu başlığı ve shell link lnk yazımızı sonlandıralım. İlk olarak KnownFolderDataBlock bölümü açık olan bir lnk sample'ı parse edelim:  

![known_folder_data_block_01.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/known_folder_data_block_01.png)

Daha sonra msdn'in paylaştığı [KNOWNFOLDERID (Knownfolders.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/knownfolderid#constants) sayfasında bulduğumuz GUID değerini burada arayalım:

![known_folder_data_block_02.png](/assets/img/2023-05-15-shell-link-lnk-binary-dosya-formatı/known_folder_data_block_02.png)

Eğer burada anlatılan bilgilerden daha büyük ileri şeyler yapmak isteyenleriniz varsa Shlobj.h dosyasını incelemelerini tavsiye ediyorum. Ayrıca öğrendiklerinizi daha da pekiştirmek için LEcmd veya farklı parser'lar kullanarak alıştırmalar yapınız. Veya belki yazının başında gösterdiğim lnk parser'ı bile geliştirmeyi deneyebilirsiniz.

Buralara kadar okuyup geldiğiniz için teşekkür ederim, umarım sizler için faydalı bir içerik olmuştur. Bir sonraki post'ta görüşmek üzere, saygı ve sevgilerimle..

# Kullanılabilir Kaynaklar

Yazı boyunca öğrendiğiniz bilgilere bilgi katmak için [P4WN3R \| Resources](/resources/#lnk-structure) kısmındaki kaynakları inceleyebilirsiniz.

# Kaynakça ve Referanslar

1. [Code pages - Globalization \| Microsoft Learn](https://learn.microsoft.com/en-us/globalization/encoding/code-pages)

2. [winsdk-10/KnownFolders.h at master · tpn/winsdk-10 · GitHub](https://github.com/tpn/winsdk-10/blob/master/Include/10.0.14393.0/um/KnownFolders.h)

3. [CSIDL (Shlobj.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/csidl)

4. [The Windows Shell - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/_shell/)

5. [Shell Links - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/links)
