---
layout: post
title: Derinlemesine Windows Processes (0x02)
categories:
- Derinlemesine Windows Serisi
- Processes
tags:
- windows
- internal
- process
- kernel-mode
- user-mode
- executive-process
- kernel-process
- process-control-block
---
Yazıya başlamadan önce çok heyecanlı olduğumu söylemek istiyorum çünkü bu yazıda process'lerin en önemli kısımlarını ele almaya çalışıcam ve artık internal'e doğru giriş yapıcaz. Yazı boyunca process'lerin internal'deki genel yapısından bahsetmeye çalışıcam. Lafı fazla uzatmadan internal'e dalmak istiyorum.

Process dediğinizde ilk başta aklınıza gelmesi gereken yapı EPROCESS (executive process) yapısıdır. Bu yapı bildiğiniz bir process'in yapısıdır. Hangi alanları hangi verileri taşır hangi alanları ne işe yarar gibi sorularınızın cevapları yazı boyunca gösterilecek.

İlk olarak EPROCESS yapısının kısa bir tanıtımını gösteren diyagramla başlayalım.

![e_process_structure_diagram_01.png](/assets/img/processes%200x02/e_process_structure_diagram_01.png)

Örnek olması için Windbg ile sistemde çalışan notepad.exe yi inceleyelim

```cpp
lkd> !process 0 0 notepad.exe
PROCESS ffffc703c06ea080
    SessionId: 31  Cid: 4088    Peb: 2dd4c3d000  ParentCid: 4720
    DirBase: 232fa8002  ObjectTable: ffffdd8ec10f2d80  HandleCount: 238.
    Image: notepad.exe

lkd> dt nt!_EPROCESS ffffc703c06ea080
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : 0x00000000`00004088 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xfffff805`12c1e0a0 - 0xffffc703`bd1144c8 ]
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : 0xd000
   +0x460 JobNotReallyActive : 0y0
   ...
```

Bu yazı boyunca bahsedeceğimiz EPROCESS yapısının önemli alanları:

- PCB (Process Control Block)

- PEB (Process Environment Block)

- VADs (Virtual Address Descriptors)

- TOKEN (Access Token)

# PCB (Process Control Block)

Bir diğer tabiriyle KPROCESS (Kernel Process) process için aşağıdaki bilgileri tutar:

- Thread'ler hakkında bilgiler

- Kullanıcı, CPU ve Kernel zamanlama bilgileri

- Context Switch bilgisi

- Process Affinity bilgisi

- Diğer...

Aşağıda KPROCESS'in C/C++ yapısını görebilirsiniz:

```cpp
//0x438 bytes (sizeof)
struct _KPROCESS
{
    struct _DISPATCHER_HEADER Header;                                       //0x0
    struct _LIST_ENTRY ProfileListHead;                                     //0x18
    ULONGLONG DirectoryTableBase;                                           //0x28
    struct _LIST_ENTRY ThreadListHead;                                      //0x30
    ULONG ProcessLock;                                                      //0x40
    ULONG ProcessTimerDelay;                                                //0x44
    ULONGLONG DeepFreezeStartTime;                                          //0x48
    struct _KAFFINITY_EX Affinity;                                          //0x50
    ULONGLONG AffinityPadding[12];                                          //0xf8
    struct _LIST_ENTRY ReadyListHead;                                       //0x158
    struct _SINGLE_LIST_ENTRY SwapListEntry;                                //0x168
    volatile struct _KAFFINITY_EX ActiveProcessors;                         //0x170
    ULONGLONG ActiveProcessorsPadding[12];                                  //0x218
    union
    {
        struct
        {
            ULONG AutoAlignment:1;                                          //0x278
            ULONG DisableBoost:1;                                           //0x278
            ULONG DisableQuantum:1;                                         //0x278
            ULONG DeepFreeze:1;                                             //0x278
            ULONG TimerVirtualization:1;                                    //0x278
            ULONG CheckStackExtents:1;                                      //0x278
            ULONG CacheIsolationEnabled:1;                                  //0x278
            ULONG PpmPolicy:3;                                              //0x278
            ULONG VaSpaceDeleted:1;                                         //0x278
            ULONG ReservedFlags:21;                                         //0x278
        };
        volatile LONG ProcessFlags;                                         //0x278
    };
    ULONG ActiveGroupsMask;                                                 //0x27c
    CHAR BasePriority;                                                      //0x280
    CHAR QuantumReset;                                                      //0x281
    CHAR Visited;                                                           //0x282
    union _KEXECUTE_OPTIONS Flags;                                          //0x283
    USHORT ThreadSeed[20];                                                  //0x284
    USHORT ThreadSeedPadding[12];                                           //0x2ac
    USHORT IdealProcessor[20];                                              //0x2c4
    USHORT IdealProcessorPadding[12];                                       //0x2ec
    USHORT IdealNode[20];                                                   //0x304
    USHORT IdealNodePadding[12];                                            //0x32c
    USHORT IdealGlobalNode;                                                 //0x344
    USHORT Spare1;                                                          //0x346
    unionvolatile _KSTACK_COUNT StackCount;                                 //0x348
    struct _LIST_ENTRY ProcessListEntry;                                    //0x350
    ULONGLONG CycleTime;                                                    //0x360
    ULONGLONG ContextSwitches;                                              //0x368
    struct _KSCHEDULING_GROUP* SchedulingGroup;                             //0x370
    ULONG FreezeCount;                                                      //0x378
    ULONG KernelTime;                                                       //0x37c
    ULONG UserTime;                                                         //0x380
    ULONG ReadyTime;                                                        //0x384
    ULONGLONG UserDirectoryTableBase;                                       //0x388
    UCHAR AddressPolicy;                                                    //0x390
    UCHAR Spare2[71];                                                       //0x391
    VOID* InstrumentationCallback;                                          //0x3d8
    union
    {
        ULONGLONG SecureHandle;                                             //0x3e0
        struct
        {
            ULONGLONG SecureProcess:1;                                      //0x3e0
            ULONGLONG Unused:1;                                             //0x3e0
        } Flags;                                                            //0x3e0
    } SecureState;                                                          //0x3e0
    ULONGLONG KernelWaitTime;                                               //0x3e8
    ULONGLONG UserWaitTime;                                                 //0x3f0
    ULONGLONG EndPadding[8];                                                //0x3f8
}; 
```

# PEB (Process Environment Block)

Diğer EPROCESS alanlarından farklı olarak kullanıcı modundan da erişlebilirdir. Process için aşağıdaki bilgileri tutar:

- Image hakkında bilgiler

- Modüller hakkında bilgiler

- Process parametreleri

- Diğer...

```cpp
//0x7c8 bytes (sizeof)
struct _PEB
{
    UCHAR InheritedAddressSpace;                                            //0x0
    UCHAR ReadImageFileExecOptions;                                         //0x1
    UCHAR BeingDebugged;                                                    //0x2
    union
    {
        UCHAR BitField;                                                     //0x3
        struct
        {
            UCHAR ImageUsesLargePages:1;                                    //0x3
            UCHAR IsProtectedProcess:1;                                     //0x3
            UCHAR IsImageDynamicallyRelocated:1;                            //0x3
            UCHAR SkipPatchingUser32Forwarders:1;                           //0x3
            UCHAR IsPackagedProcess:1;                                      //0x3
            UCHAR IsAppContainer:1;                                         //0x3
            UCHAR IsProtectedProcessLight:1;                                //0x3
            UCHAR IsLongPathAwareProcess:1;                                 //0x3
        };
    };
    UCHAR Padding0[4];                                                      //0x4
    VOID* Mutant;                                                           //0x8
    VOID* ImageBaseAddress;                                                 //0x10
    struct _PEB_LDR_DATA* Ldr;                                              //0x18
    struct _RTL_USER_PROCESS_PARAMETERS* ProcessParameters;                 //0x20
    VOID* SubSystemData;                                                    //0x28
    VOID* ProcessHeap;                                                      //0x30
    struct _RTL_CRITICAL_SECTION* FastPebLock;                              //0x38
    union _SLIST_HEADER* volatile AtlThunkSListPtr;                         //0x40
    VOID* IFEOKey;                                                          //0x48
    union
    {
        ULONG CrossProcessFlags;                                            //0x50
        struct
        {
            ULONG ProcessInJob:1;                                           //0x50
            ULONG ProcessInitializing:1;                                    //0x50
            ULONG ProcessUsingVEH:1;                                        //0x50
            ULONG ProcessUsingVCH:1;                                        //0x50
            ULONG ProcessUsingFTH:1;                                        //0x50
            ULONG ProcessPreviouslyThrottled:1;                             //0x50
            ULONG ProcessCurrentlyThrottled:1;                              //0x50
            ULONG ProcessImagesHotPatched:1;                                //0x50
            ULONG ReservedBits0:24;                                         //0x50
        };
    };
    UCHAR Padding1[4];                                                      //0x54
    union
    {
        VOID* KernelCallbackTable;                                          //0x58
        VOID* UserSharedInfoPtr;                                            //0x58
    };
    ULONG SystemReserved;                                                   //0x60
    ULONG AtlThunkSListPtr32;                                               //0x64
    VOID* ApiSetMap;                                                        //0x68
    ULONG TlsExpansionCounter;                                              //0x70
    UCHAR Padding2[4];                                                      //0x74
    VOID* TlsBitmap;                                                        //0x78
    ULONG TlsBitmapBits[2];                                                 //0x80
    VOID* ReadOnlySharedMemoryBase;                                         //0x88
    VOID* SharedData;                                                       //0x90
    VOID** ReadOnlyStaticServerData;                                        //0x98
    VOID* AnsiCodePageData;                                                 //0xa0
    VOID* OemCodePageData;                                                  //0xa8
    VOID* UnicodeCaseTableData;                                             //0xb0
    ULONG NumberOfProcessors;                                               //0xb8
    ULONG NtGlobalFlag;                                                     //0xbc
    union _LARGE_INTEGER CriticalSectionTimeout;                            //0xc0
    ULONGLONG HeapSegmentReserve;                                           //0xc8
    ULONGLONG HeapSegmentCommit;                                            //0xd0
    ULONGLONG HeapDeCommitTotalFreeThreshold;                               //0xd8
    ULONGLONG HeapDeCommitFreeBlockThreshold;                               //0xe0
    ULONG NumberOfHeaps;                                                    //0xe8
    ULONG MaximumNumberOfHeaps;                                             //0xec
    VOID** ProcessHeaps;                                                    //0xf0
    VOID* GdiSharedHandleTable;                                             //0xf8
    VOID* ProcessStarterHelper;                                             //0x100
    ULONG GdiDCAttributeList;                                               //0x108
    UCHAR Padding3[4];                                                      //0x10c
    struct _RTL_CRITICAL_SECTION* LoaderLock;                               //0x110
    ULONG OSMajorVersion;                                                   //0x118
    ULONG OSMinorVersion;                                                   //0x11c
    USHORT OSBuildNumber;                                                   //0x120
    USHORT OSCSDVersion;                                                    //0x122
    ULONG OSPlatformId;                                                     //0x124
    ULONG ImageSubsystem;                                                   //0x128
    ULONG ImageSubsystemMajorVersion;                                       //0x12c
    ULONG ImageSubsystemMinorVersion;                                       //0x130
    UCHAR Padding4[4];                                                      //0x134
    ULONGLONG ActiveProcessAffinityMask;                                    //0x138
    ULONG GdiHandleBuffer[60];                                              //0x140
    VOID (*PostProcessInitRoutine)();                                       //0x230
    VOID* TlsExpansionBitmap;                                               //0x238
    ULONG TlsExpansionBitmapBits[32];                                       //0x240
    ULONG SessionId;                                                        //0x2c0
    UCHAR Padding5[4];                                                      //0x2c4
    union _ULARGE_INTEGER AppCompatFlags;                                   //0x2c8
    union _ULARGE_INTEGER AppCompatFlagsUser;                               //0x2d0
    VOID* pShimData;                                                        //0x2d8
    VOID* AppCompatInfo;                                                    //0x2e0
    struct _UNICODE_STRING CSDVersion;                                      //0x2e8
    struct _ACTIVATION_CONTEXT_DATA* ActivationContextData;                 //0x2f8
    struct _ASSEMBLY_STORAGE_MAP* ProcessAssemblyStorageMap;                //0x300
    struct _ACTIVATION_CONTEXT_DATA* SystemDefaultActivationContextData;    //0x308
    struct _ASSEMBLY_STORAGE_MAP* SystemAssemblyStorageMap;                 //0x310
    ULONGLONG MinimumStackCommit;                                           //0x318
    VOID* SparePointers[4];                                                 //0x320
    ULONG SpareUlongs[5];                                                   //0x340
    VOID* WerRegistrationData;                                              //0x358
    VOID* WerShipAssertPtr;                                                 //0x360
    VOID* pUnused;                                                          //0x368
    VOID* pImageHeaderHash;                                                 //0x370
    union
    {
        ULONG TracingFlags;                                                 //0x378
        struct
        {
            ULONG HeapTracingEnabled:1;                                     //0x378
            ULONG CritSecTracingEnabled:1;                                  //0x378
            ULONG LibLoaderTracingEnabled:1;                                //0x378
            ULONG SpareTracingBits:29;                                      //0x378
        };
    };
    UCHAR Padding6[4];                                                      //0x37c
    ULONGLONG CsrServerReadOnlySharedMemoryBase;                            //0x380
    ULONGLONG TppWorkerpListLock;                                           //0x388
    struct _LIST_ENTRY TppWorkerpList;                                      //0x390
    VOID* WaitOnAddressHashTable[128];                                      //0x3a0
    VOID* TelemetryCoverageHeader;                                          //0x7a0
    ULONG CloudFileFlags;                                                   //0x7a8
    ULONG CloudFileDiagFlags;                                               //0x7ac
    CHAR PlaceholderCompatibilityMode;                                      //0x7b0
    CHAR PlaceholderCompatibilityModeReserved[7];                           //0x7b1
    struct _LEAP_SECOND_DATA* LeapSecondData;                               //0x7b8
    union
    {
        ULONG LeapSecondFlags;                                              //0x7c0
        struct
        {
            ULONG SixtySecondEnabled:1;                                     //0x7c0
            ULONG Reserved:31;                                              //0x7c0
        };
    };
    ULONG NtGlobalFlag2;                                                    //0x7c4
}; 
```

Bu yapıdaki tüm alanları öğrenmemize **şu anlık** gerek duymuyorum. Eğer amacınız kernel development veya kernel exploit development gibi alanlarda ilerlemekse bişey diyemem. Bu yazı daha çok malware analizi için process internaline giriş yazısı gibi olacak. Yani eğer malware geliştirirken veya malware analizi için debugger vs. kullanırken bu yazıdaki bilgiler kesinlikle karşınıza çıkacak ve sizinde bu konularda yabancı kalmanızı hiç istemem :)

Ele alacağımız alanlar basitten karmaşığa doğru:

**ImageFile:** Process'e yüklenmiş olan PE dosyasıdır.

**~~ProcessParameters~~:** 

**CommandLine:** PE dosyasının çalıştırılması için arka planda hangi komut verilmişse aynısı buraya gelir. Örneğin *notepad.exe -a* dersek bu kısımdaki değer aynen öyle olur.

**CurrentDirectory:** Process oluşumunu sağlayan CreateProcess fonksiyonunun parametresi olan current directory'e hangi değer girilirse buraya yazılır (genelde Null bırakıyoruz ama böyle olunca da defaultta home dizini yazılıyor diye biliyorum).

**ImageBaseAddress:** Process'e yüklenmiş olan (memory'deki) PE dosyasının hafıza adresini tutar. Burası process için önemli bir alandır.

**Environment:** Process için gerekli olan çevre değişkenleri burada tutulur. Yine CreateProcess ile ek değerler eklenilebilir.

```
...
PUBLIC=C:\Users\Public
SESSIONNAME=Console
SystemDrive=C:
SystemRoot=C:\Windows
TEMP=C:\Users\user\AppData\Local\Temp
TMP=C:\Users\user\AppData\Local\Temp
...
```

**Ldr:** Process'in en önemli kısımlarından birisidir, process için hafızaya yüklenen modülleri gösterir. Bu alan double linked list veri yapısıyla tasarlanmıştır. Yani aşağıdaki mantıkla ilerler:

![double_linked_list_01.png](double_linked_list_01.png)

Ldr alanı _PEB_LDR_DATA yapısındadır ve bu alanı windbg ile aşağıdaki gibi incelersek:

```cpp
lkd> dt nt!_PEB_LDR_DATA 00007ffde65dc4c0
   +0x000 Length           : 0x58
   +0x004 Initialized      : 0x1 ''
   +0x008 SsHandle         : (null) 
   +0x010 InLoadOrderModuleList : _LIST_ENTRY [ 0x00000178`06623240 - 0x00000178`0665e7f0 ]
   +0x020 InMemoryOrderModuleList : _LIST_ENTRY [ 0x00000178`06623250 - 0x00000178`0665e800 ]
   +0x030 InInitializationOrderModuleList : _LIST_ENTRY [ 0x00000178`066230d0 - 0x00000178`0665e810 ]
   +0x040 EntryInProgress  : (null) 
   +0x048 ShutdownInProgress : 0 ''
   +0x050 ShutdownThreadId : (null) 
```

Üç adet önemli alanı ele alacağımızı fark etmişsinizdir. Bu alanların her birisi _LIST_ENTRY yapısındadır ve 

```cpp
dt nt!_PEB_LDR_DATA 00007ffde65dc4c0
dt nt!_LDR_DATA_TABLE_ENTRY 0x178066230b0
```

Direkt olarak !peb komutuyla modülleri listeleyebiliriz

# VADs (Virtual Address Descriptors)

dsadsa

# TOKEN (Access Token)

EPROCESS objesi için güvenlik ayarlarını barındırır.

| EPROCESS'teki Alan                               | Açıklaması                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kernel process (KPROCESS) block                  | Common dispatcher object header, pointer to the process page directory, list of kernel thread (KTHREAD) blocks belonging to the process, default base priority, affinity mask, and total kernel and user time and CPU clock cycles for the threads in the process.                                                                                 |
| Process Identification                           | Process id numarası (windbg'de Cid diye geçer)                                                                                                                                                                                                                                                                                                     |
| Quota Block                                      | Limits on processor usage, nonpaged pool, paged pool, and page file usage plus current and peak process nonpaged and paged pool usage. (Note: Several processes can share this structure: all the system processes in session 0 point to a single systemwide quota block; all other processes in interactive sessions share a single quota block.) |
| Virtual Address Descriptors (VADs)               | Series of data structures that describes the status of the portions of the address space that exist in the process.                                                                                                                                                                                                                                |
| Working Set Information                          | Pointer to working set list (MMWSL structure); current, peak, minimum, and maximum working set size; last trim time; page fault count; memory priority; outswap flags; page fault history.                                                                                                                                                         |
| Virtual Memory Information                       | Current and peak virtual size, page file usage, hardware page table entry for process page directory.                                                                                                                                                                                                                                              |
| Exception legacy local procedure call (LPC) port | Interprocess communication channel to which the process manager sends a message when one of the process’s threads causes an exception.                                                                                                                                                                                                             |
| Debugging object                                 | Executive object through which the user-mode debugging infrastructure sends notifications when one of the process’s threads causes a debug event.                                                                                                                                                                                                  |
| Access token (TOKEN)                             | Executive object describing the security profile of this process.                                                                                                                                                                                                                                                                                  |
| Handle table                                     | Process üzerinde çalışan objelerin handle pointer'ları                                                                                                                                                                                                                                                                                             |
| Device map                                       | Address of object directory to resolve device name references in (supports multiple users).                                                                                                                                                                                                                                                        |
| Process environment block (PEB)                  | Process'e yüklenen image hakkında bilgiler tutar                                                                                                                                                                                                                                                                                                   |
| Windows subsystem process block (W32PROCESS)     | Windows sub-system'ın kernel modu tarafından gereken process ayrıntıları buradadır                                                                                                                                                                                                                                                                 |

dasdsa

# Kaynakça ve Referanslar

1. Veriyapıları ve Algoritmalar, Toros Rıfat Çölkesen
