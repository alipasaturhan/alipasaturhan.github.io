---
layout: post
title: Derinlemesine Windows Processes (0x02)
date: 2023-06-02 17:43 +0300
categories: [Derinlemesine Windows Serisi, Processes]
tags: [windows, internal, process, kernel-mode, user-mode, executive-process, kernel-process, process-control-block]
---

Yazıya başlamadan önce çok heyecanlı olduğumu söylemek istiyorum çünkü bu yazıda process'lerin en önemli kısımlarını ele almaya çalışıcam ve artık internal'e doğru giriş yapıcaz. Yazı boyunca process'lerin internal'deki genel yapısından bahsetmeye çalışıcam. Lafı fazla uzatmadan internal'e dalmak istiyorum.

Process dediğinizde ilk başta aklınıza gelmesi gereken yapı EPROCESS (executive process) yapısıdır. Bu yapı bildiğiniz bir process'in yapısıdır. Hangi alanları hangi verileri taşır hangi alanları ne işe yarar gibi sorularınızın cevapları yazı boyunca gösterilecek.

İlk olarak EPROCESS yapısının kısa bir tanıtımını gösteren diyagramla başlayalım.

![e_process_structure_diagram_01.png](/assets/img/2023-06-02-derinlemesine-windows-processes-0x02/e_process_structure_diagram_01.png)

Örnek olması için Windbg ile sistemde çalışan notepad.exe yi inceleyelim

```cpp
lkd> !process 0 0 notepad.exe
PROCESS ffffcd0685106080
    SessionId: 31  Cid: 4088    Peb: 2dd4c3d000  ParentCid: 4720
    DirBase: 232fa8002  ObjectTable: ffffdd8ec10f2d80  HandleCount: 238.
    Image: notepad.exe

lkd> dt nt!_EPROCESS ffffcd0685106080
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : 0x00000000`0000453c Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY [ 0xffffcd06`859d14c8 - 0xffffcd06`87cce4c8 ]
   +0x458 RundownProtect   : _EX_RUNDOWN_REF
   +0x460 Flags2           : 0xd000
   +0x460 JobNotReallyActive : 0y0
   +0x460 AccountingFolded : 0y0
   +0x460 NewProcessReported : 0y0
   +0x460 ExitProcessReported : 0y0
   +0x460 ReportCommitChanges : 0y0
   +0x460 LastReportMemory : 0y0
   +0x460 ForceWakeCharge  : 0y0
   +0x460 CrossSessionCreate : 0y0
   +0x460 NeedsHandleRundown : 0y0
   +0x460 RefTraceEnabled  : 0y0
   +0x460 PicoCreated      : 0y0
   +0x460 EmptyJobEvaluated : 0y0
   +0x460 DefaultPagePriority : 0y101
   +0x460 PrimaryTokenFrozen : 0y1
   +0x460 ProcessVerifierTarget : 0y0
   +0x460 RestrictSetThreadContext : 0y0
   +0x460 AffinityPermanent : 0y0
   +0x460 AffinityUpdateEnable : 0y0
   +0x460 PropagateNode    : 0y0
   +0x460 ExplicitAffinity : 0y0
   +0x460 ProcessExecutionState : 0y00
   +0x460 EnableReadVmLogging : 0y0
   +0x460 EnableWriteVmLogging : 0y0
   +0x460 FatalAccessTerminationRequested : 0y0
   +0x460 DisableSystemAllowedCpuSet : 0y0
   +0x460 ProcessStateChangeRequest : 0y00
   +0x460 ProcessStateChangeInProgress : 0y0
   +0x460 InPrivate        : 0y0
   +0x464 Flags            : 0x144d0c01
   +0x464 CreateReported   : 0y1
   +0x464 NoDebugInherit   : 0y0
   +0x464 ProcessExiting   : 0y0
   +0x464 ProcessDelete    : 0y0
   +0x464 ManageExecutableMemoryWrites : 0y0
   +0x464 VmDeleted        : 0y0
   +0x464 OutswapEnabled   : 0y0
   +0x464 Outswapped       : 0y0
   +0x464 FailFastOnCommitFail : 0y0
   +0x464 Wow64VaSpace4Gb  : 0y0
   +0x464 AddressSpaceInitialized : 0y11
   +0x464 SetTimerResolution : 0y0
   +0x464 BreakOnTermination : 0y0
   +0x464 DeprioritizeViews : 0y0
   +0x464 WriteWatch       : 0y0
   +0x464 ProcessInSession : 0y1
   +0x464 OverrideAddressSpace : 0y0
   +0x464 HasAddressSpace  : 0y1
   +0x464 LaunchPrefetched : 0y1
   +0x464 Background       : 0y0
   +0x464 VmTopDown        : 0y0
   +0x464 ImageNotifyDone  : 0y1
   +0x464 PdeUpdateNeeded  : 0y0
   +0x464 VdmAllowed       : 0y0
   +0x464 ProcessRundown   : 0y0
   +0x464 ProcessInserted  : 0y1
   +0x464 DefaultIoPriority : 0y010
   +0x464 ProcessSelfDelete : 0y0
   +0x464 SetTimerResolutionLink : 0y0
   +0x468 CreateTime       : _LARGE_INTEGER 0x01d9a0eb`ed2820eb
   +0x470 ProcessQuotaUsage : [2] 0x3260
   +0x480 ProcessQuotaPeak : [2] 0x3618
   +0x490 PeakVirtualSize  : 0x00000201`09dc4000
   +0x498 VirtualSize      : 0x00000201`092db000
   +0x4a0 SessionProcessLinks : _LIST_ENTRY [ 0xffffcd06`859d1520 - 0xffffcd06`87cce520 ]
   +0x4b0 ExceptionPortData : 0xffffcd06`7df10df0 Void
   +0x4b0 ExceptionPortValue : 0xffffcd06`7df10df0
   +0x4b0 ExceptionPortState : 0y000
   +0x4b8 Token            : _EX_FAST_REF
   +0x4c0 MmReserved       : 0
   +0x4c8 AddressCreationLock : _EX_PUSH_LOCK
   +0x4d0 PageTableCommitmentLock : _EX_PUSH_LOCK
   +0x4d8 RotateInProgress : (null) 
   +0x4e0 ForkInProgress   : (null) 
   +0x4e8 CommitChargeJob  : (null) 
   +0x4f0 CloneRoot        : _RTL_AVL_TREE
   +0x4f8 NumberOfPrivatePages : 0x19a
   +0x500 NumberOfLockedPages : 0
   +0x508 Win32Process     : 0xfffff1d1`8827d010 Void
   +0x510 Job              : (null) 
   +0x518 SectionObject    : 0xffffdf01`22c8ad90 Void
   +0x520 SectionBaseAddress : 0x00007ff6`09c50000 Void
   +0x528 Cookie           : 0xac584df1
   +0x530 WorkingSetWatch  : (null) 
   +0x538 Win32WindowStation : 0x00000000`000000a8 Void
   +0x540 InheritedFromUniqueProcessId : 0x00000000`00004928 Void
   +0x548 OwnerProcessId   : 0x492a
   +0x550 Peb              : 0x00000004`d1af4000 _PEB
   +0x558 Session          : 0xffffbb81`3a957000 _MM_SESSION_SPACE
   +0x560 Spare1           : (null) 
   +0x568 QuotaBlock       : 0xffffcd06`80f29d40 _EPROCESS_QUOTA_BLOCK
   +0x570 ObjectTable      : 0xffffdf01`2820dac0 _HANDLE_TABLE
   +0x578 DebugPort        : (null) 
   +0x580 WoW64Process     : (null) 
   +0x588 DeviceMap        : 0xffffdf01`310ce720 Void
   +0x590 EtwDataSource    : 0xffffcd06`8a6f4c50 Void
   +0x598 PageDirectoryPte : 0
   +0x5a0 ImageFilePointer : 0xffffcd06`95c4a710 _FILE_OBJECT
   +0x5a8 ImageFileName    : [15]  "notepad.exe"
   +0x5b7 PriorityClass    : 0x2 ''
   +0x5b8 SecurityPort     : (null) 
   +0x5c0 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x5c8 JobLinks         : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   +0x5d8 HighestUserAddress : 0x00007fff`ffff0000 Void
   +0x5e0 ThreadListHead   : _LIST_ENTRY [ 0xffffcd06`8a4e3728 - 0xffffcd06`8a4e3728 ]
   +0x5f0 ActiveThreads    : 1
   +0x5f4 ImagePathHash    : 0x9fb27c0e
   +0x5f8 DefaultHardErrorProcessing : 1
   +0x5fc LastThreadExitStatus : 0n0
   +0x600 PrefetchTrace    : _EX_FAST_REF
   +0x608 LockedPagesList  : (null) 
   +0x610 ReadOperationCount : _LARGE_INTEGER 0x0
   +0x618 WriteOperationCount : _LARGE_INTEGER 0x0
   +0x620 OtherOperationCount : _LARGE_INTEGER 0x1c
   +0x628 ReadTransferCount : _LARGE_INTEGER 0x0
   +0x630 WriteTransferCount : _LARGE_INTEGER 0x0
   +0x638 OtherTransferCount : _LARGE_INTEGER 0x0
   +0x640 CommitChargeLimit : 0
   +0x648 CommitCharge     : 0x29e
   +0x650 CommitChargePeak : 0x305
   +0x680 Vm               : _MMSUPPORT_FULL
   +0x7c0 MmProcessLinks   : _LIST_ENTRY [ 0xffffcd06`859d1840 - 0xffffcd06`87cce840 ]
   +0x7d0 ModifiedPageCount : 0x142
   +0x7d4 ExitStatus       : 0n259
   +0x7d8 VadRoot          : _RTL_AVL_TREE
   +0x7e0 VadHint          : 0xffffcd06`8d0f2d80 Void
   +0x7e8 VadCount         : 0x5b
   +0x7f0 VadPhysicalPages : 0
   +0x7f8 VadPhysicalPagesLimit : 0
   +0x800 AlpcContext      : _ALPC_PROCESS_CONTEXT
   +0x820 TimerResolutionLink : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   +0x830 TimerResolutionStackRecord : (null) 
   +0x838 RequestedTimerResolution : 0
   +0x83c SmallestTimerResolution : 0
   +0x840 ExitTime         : _LARGE_INTEGER 0x0
   +0x848 InvertedFunctionTable : (null) 
   +0x850 InvertedFunctionTableLock : _EX_PUSH_LOCK
   +0x858 ActiveThreadsHighWatermark : 7
   +0x85c LargePrivateVadCount : 0
   +0x860 ThreadListLock   : _EX_PUSH_LOCK
   +0x868 WnfContext       : 0xffffdf01`22cb23e0 Void
   +0x870 ServerSilo       : (null) 
   +0x878 SignatureLevel   : 0 ''
   +0x879 SectionSignatureLevel : 0 ''
   +0x87a Protection       : _PS_PROTECTION
   +0x87b HangCount        : 0y000
   +0x87b GhostCount       : 0y000
   +0x87b PrefilterException : 0y0
   +0x87c Flags3           : 0x40c000
   +0x87c Minimal          : 0y0
   +0x87c ReplacingPageRoot : 0y0
   +0x87c Crashed          : 0y0
   +0x87c JobVadsAreTracked : 0y0
   +0x87c VadTrackingDisabled : 0y0
   +0x87c AuxiliaryProcess : 0y0
   +0x87c SubsystemProcess : 0y0
   +0x87c IndirectCpuSets  : 0y0
   +0x87c RelinquishedCommit : 0y0
   +0x87c HighGraphicsPriority : 0y0
   +0x87c CommitFailLogged : 0y0
   +0x87c ReserveFailLogged : 0y0
   +0x87c SystemProcess    : 0y0
   +0x87c HideImageBaseAddresses : 0y0
   +0x87c AddressPolicyFrozen : 0y1
   +0x87c ProcessFirstResume : 0y1
   +0x87c ForegroundExternal : 0y0
   +0x87c ForegroundSystem : 0y0
   +0x87c HighMemoryPriority : 0y0
   +0x87c EnableProcessSuspendResumeLogging : 0y0
   +0x87c EnableThreadSuspendResumeLogging : 0y0
   +0x87c SecurityDomainChanged : 0y0
   +0x87c SecurityFreezeComplete : 0y1
   +0x87c VmProcessorHost  : 0y0
   +0x87c VmProcessorHostTransition : 0y0
   +0x87c AltSyscall       : 0y0
   +0x87c TimerResolutionIgnore : 0y0
   +0x87c DisallowUserTerminate : 0y0
   +0x880 DeviceAsid       : 0n0
   +0x888 SvmData          : (null) 
   +0x890 SvmProcessLock   : _EX_PUSH_LOCK
   +0x898 SvmLock          : 0
   +0x8a0 SvmProcessDeviceListHead : _LIST_ENTRY [ 0xffffcd06`85106920 - 0xffffcd06`85106920 ]
   +0x8b0 LastFreezeInterruptTime : 0
   +0x8b8 DiskCounters     : 0xffffcd06`85106ac0 _PROCESS_DISK_COUNTERS
   +0x8c0 PicoContext      : (null) 
   +0x8c8 EnclaveTable     : (null) 
   +0x8d0 EnclaveNumber    : 0
   +0x8d8 EnclaveLock      : _EX_PUSH_LOCK
   +0x8e0 HighPriorityFaultsAllowed : 0
   +0x8e8 EnergyContext    : 0xffffcd06`85106ae8 _PO_PROCESS_ENERGY_CONTEXT
   +0x8f0 VmContext        : (null) 
   +0x8f8 SequenceNumber   : 0x5df9
   +0x900 CreateInterruptTime : 0x000001ef`a2043c64
   +0x908 CreateUnbiasedInterruptTime : 0x000000cf`fbfc05c6
   +0x910 TotalUnbiasedFrozenTime : 0
   +0x918 LastAppStateUpdateTime : 0x00000313`ced7fe9d
   +0x920 LastAppStateUptime : 0y0000000000000000000001000101010001001100110111001100010000111 (0x8a899b9887)
   +0x920 LastAppState     : 0y101
   +0x928 SharedCommitCharge : 0x4c1
   +0x930 SharedCommitLock : _EX_PUSH_LOCK
   +0x938 SharedCommitLinks : _LIST_ENTRY [ 0xffffdf01`154416d8 - 0xffffdf01`1b8420f8 ]
   +0x948 AllowedCpuSets   : 0
   +0x950 DefaultCpuSets   : 0
   +0x948 AllowedCpuSetsIndirect : (null) 
   +0x950 DefaultCpuSetsIndirect : (null) 
   +0x958 DiskIoAttribution : (null) 
   +0x960 DxgProcess       : 0xffffdf01`1aec15e0 Void
   +0x968 Win32KFilterSet  : 0
   +0x970 ProcessTimerDelay : _PS_INTERLOCKED_TIMER_DELAY_VALUES
   +0x978 KTimerSets       : 0
   +0x97c KTimer2Sets      : 0
   +0x980 ThreadTimerSets  : 4
   +0x988 VirtualTimerListLock : 0
   +0x990 VirtualTimerListHead : _LIST_ENTRY [ 0xffffcd06`85106a10 - 0xffffcd06`85106a10 ]
   +0x9a0 WakeChannel      : _WNF_STATE_NAME
   +0x9a0 WakeInfo         : _PS_PROCESS_WAKE_INFORMATION
   +0x9d0 MitigationFlags  : 0x21
   +0x9d0 MitigationFlagsValues : <anonymous-tag>
   +0x9d4 MitigationFlags2 : 0x40000000
   +0x9d4 MitigationFlags2Values : <anonymous-tag>
   +0x9d8 PartitionObject  : 0xffffcd06`74aaca40 Void
   +0x9e0 SecurityDomain   : 0x00000001`00000097
   +0x9e8 ParentSecurityDomain : 0x00000001`00000097
   +0x9f0 CoverageSamplerContext : (null) 
   +0x9f8 MmHotPatchContext : (null) 
   +0xa00 DynamicEHContinuationTargetsTree : _RTL_AVL_TREE
   +0xa08 DynamicEHContinuationTargetsLock : _EX_PUSH_LOCK
   +0xa10 DynamicEnforcedCetCompatibleRanges : _PS_DYNAMIC_ENFORCED_ADDRESS_RANGES
   +0xa20 DisabledComponentFlags : 0
   +0xa28 PathRedirectionHashes : (null) 
```

Bu yazı boyunca bahsedeceğimiz EPROCESS yapısının önemli alanları:

- PCB (Process Control Block)

- PEB (Process Environment Block)

Bir sonraki part'ta ise aşağıdaki bölümlerden bahsetmeye çalışacam.

- VADs (Virtual Address Descriptors)

- TOKEN (Access Token)

# PCB (Process Control Block)

Bir diğer tabiriyle KPROCESS (Kernel Process) process için aşağıdaki bilgileri tutar:

- Thread'ler hakkında bilgiler

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

KPROCESS yapısındaki diğer alanları merak ediyorsanız, bildiğiniz cpu'nun çalışması için gördüğü genel bilgileri görürsünüz. Yani affinitiy mask bilgisi, timing'ler (user, cpu, kernel, ...), context switch vs. cpu'yu ilgilendiren alanlardır. Biz şu anlık malware analizleri için process internaline giriş yapıyoruz.

Bu başlıkta process'te olan thread'lerin nerede saklandığını ve nasıl bulunduğunu göstermeye çalışıcam. Aslında thread'ler için ayrı bir post paylaşmayı düşünüyordum, bu başlıkla beraber şimdiden thread internal'inden çok kısa bahsetmiş olacam.

Thread'ler EPROCESS yapısındaki Pcb->ThreadList'te saklanır ve LIST_ENTRY yani doubly linked şeklinde görüntülenir.

```cpp
lkd> dt nt!_EPROCESS ffffcd0683883080 -y pcb
   +0x000 Pcb : _KPROCESS
lkd> dt nt!_KPROCESS ffffcd0683883080 -y threadlist
   +0x030 ThreadListHead : _LIST_ENTRY [ 0xffffcd06`883eb378 - 0xffffcd06`883eb378 ]
lkd> dl 0xffffcd06`883eb378
ffffcd06`883eb378  ffffcd06`838830b0 ffffcd06`838830b0
ffffcd06`883eb388  ffffcd06`883eb388 ffffcd06`883eb388
ffffcd06`838830b0  ffffcd06`883eb378 ffffcd06`883eb378
ffffcd06`838830c0  00000000`00000000 00000000`00000000
```

Her bir entry için _ETHREAD yapısı gösterilirse process'te varolan thread'ler kolayca listelenebilir. Bununla uğraşmak istemezseniz direkt !list veya !process komutlarını kullanarak listeleyebilirsiniz.

> LIST_ENTRY yapısını PEB başlığında Ldr'ları anlatırken kısaca bahsettim. Bu yapı kaba tabiriyle dinamik verileri sırasıyla göstermeye yarıyor diyebiliriz.

# PEB (Process Environment Block)

Diğer EPROCESS alanlarından farklı olarak kullanıcı modundan da erişlebilirdir. Process için aşağıdaki bilgileri tutar:

- Image hakkında bilgiler

- Modüller hakkında bilgiler

- Process parametreleri

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

Bu yapıdaki tüm alanları öğrenmemize **şu anlık** gerek duymuyorum. Eğer amacınız kernel development veya kernel exploit development gibi alanlarda ilerlemekse bişey diyemem. Bu yazı daha çok malware analizi için process internaline giriş yazısı gibi olacak. Yani malware geliştirirken veya malware analizi için debugger vs. kullanırken bu yazıdaki bilgiler kesinlikle karşınıza çıkacak ve sizinde bu konularda yabancı kalmanızı hiç istemem :)

Ele alacağımız alanlar basitten karmaşığa doğru:

**ImageFile:** Process'e yüklenmiş olan PE dosyasıdır.

**~~ProcessParameters~~:** 

**CommandLine:** PE dosyasının çalıştırılması için arka planda hangi komut verilmişse aynısı buraya gelir. Örneğin *notepad.exe -a* dersek bu kısımdaki değer aynen öyle olur.

**CurrentDirectory:** Process oluşumunu sağlayan CreateProcess fonksiyonunun parametresi olan current directory'e hangi değer girilirse buraya yazılır (genelde Null bırakıyoruz ama böyle olunca da defaultta home dizini yazılıyor diye biliyorum).

**ImageBaseAddress:** Process'e yüklenmiş olan PE dosyasının hafıza adresini tutar. Burası process için önemli bir alandır.

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

![double_linked_list_01.png](/assets/img/2023-06-02-derinlemesine-windows-processes-0x02/double_linked_list_01.png)

Ldr alanı _PEB_LDR_DATA yapısındadır ve bu alanı windbg ile aşağıdaki gibi incelersek:

```cpp
lkd> dt nt!_EPROCESS ffffcd0685106080 -y Peb
   +0x550 Peb : 0x00000004`d1af4000 _PEB
lkd> dt nt!_PEB 0x00000004`d1af4000 -y ldr
   +0x018 Ldr : 0x00007fff`efb9c4c0 _PEB_LDR_DATA
lkd> dt nt!_PEB_LDR_DATA 0x00007fff`efb9c4c0
   +0x000 Length           : 0x58
   +0x004 Initialized      : 0x1 ''
   +0x008 SsHandle         : (null) 
   +0x010 InLoadOrderModuleList : _LIST_ENTRY [ 0x0000018e`0e8b32a0 - 0x0000018e`0e8ed250 ]
   +0x020 InMemoryOrderModuleList : _LIST_ENTRY [ 0x0000018e`0e8b32b0 - 0x0000018e`0e8ed260 ]
   +0x030 InInitializationOrderModuleList : _LIST_ENTRY [ 0x0000018e`0e8b3130 - 0x0000018e`0e8ebbe0 ]
   +0x040 EntryInProgress  : (null) 
   +0x048 ShutdownInProgress : 0 ''
   +0x050 ShutdownThreadId : (null) 
```

Üç adet önemli alanı ele alacağımızı fark etmişsinizdir. Bu alanların her birisi _LIST_ENTRY yapısı ile double linked list yapısı referans alınarak belli bir sırayla gösterilir. Sırasıyla gösterilen her entry ise LDR_DATA_TABLE_ENTRY yapısındadır.

![ldr_structure_diagram_01.png](/assets/img/2023-06-02-derinlemesine-windows-processes-0x02/ldr_structure_diagram_01.png)

InLoadOrderModuleList, InMemoryOrderModuleList ve InInitializationOrderModuleList aynı data'ları tutar. Tek farkları isminden de anlayacağınız üzere dll'leri belirli bir sıraya göre tutmalarıdır. InLoadOrderModuleList alanındandaki ilk entry'den devam etmek istersek:

```cpp
lkd> dt nt!_LDR_DATA_TABLE_ENTRY 0x0000018e`0e8b32a0
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x0000018e`0e8b3110 - 0x00007fff`efb9c4d0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x0000018e`0e8b3120 - 0x00007fff`efb9c4e0 ]
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   +0x030 DllBase          : 0x00007ff6`09c50000 Void
   +0x038 EntryPoint       : 0x00007ff6`09c73f40 Void
   +0x040 SizeOfImage      : 0x38000
   +0x048 FullDllName      : _UNICODE_STRING "C:\Windows\system32\notepad.exe"
   +0x058 BaseDllName      : _UNICODE_STRING "notepad.exe"
   +0x068 FlagGroup        : [4]  "???"
   +0x068 Flags            : 0xa2cc
   +0x068 PackagedBinary   : 0y0
   +0x068 MarkedForRemoval : 0y0
   +0x068 ImageDll         : 0y1
   +0x068 LoadNotificationsSent : 0y1
   +0x068 TelemetryEntryProcessed : 0y0
   +0x068 ProcessStaticImport : 0y0
   +0x068 InLegacyLists    : 0y1
   +0x068 InIndexes        : 0y1
   +0x068 ShimDll          : 0y0
   +0x068 InExceptionTable : 0y1
   +0x068 ReservedFlags1   : 0y00
   +0x068 LoadInProgress   : 0y0
   +0x068 LoadConfigProcessed : 0y1
   +0x068 EntryProcessed   : 0y0
   +0x068 ProtectDelayLoad : 0y1
   +0x068 ReservedFlags3   : 0y00
   +0x068 DontCallForThreads : 0y0
   +0x068 ProcessAttachCalled : 0y0
   +0x068 ProcessAttachFailed : 0y0
   +0x068 CorDeferredValidate : 0y0
   +0x068 CorImage         : 0y0
   +0x068 DontRelocate     : 0y0
   +0x068 CorILOnly        : 0y0
   +0x068 ChpeImage        : 0y0
   +0x068 ReservedFlags5   : 0y00
   +0x068 Redirected       : 0y0
   +0x068 ReservedFlags6   : 0y00
   +0x068 CompatDatabaseProcessed : 0y0
   +0x06c ObsoleteLoadCount : 0xffff
   +0x06e TlsIndex         : 0
   +0x070 HashLinks        : _LIST_ENTRY [ 0x0000018e`0e8cc060 - 0x00007fff`efb9c190 ]
   +0x080 TimeDateStamp    : 0xbdd4adcd
   +0x088 EntryPointActivationContext : (null) 
   +0x090 Lock             : (null) 
   +0x098 DdagNode         : 0x0000018e`0e8b33d0 _LDR_DDAG_NODE
   +0x0a0 NodeModuleLink   : _LIST_ENTRY [ 0x0000018e`0e8b33d0 - 0x0000018e`0e8b33d0 ]
   +0x0b0 LoadContext      : (null) 
   +0x0b8 ParentDllBase    : (null) 
   +0x0c0 SwitchBackContext : 0x00007fff`efb4d374 Void
   +0x0c8 BaseAddressIndexNode : _RTL_BALANCED_NODE
   +0x0e0 MappingInfoIndexNode : _RTL_BALANCED_NODE
   +0x0f8 OriginalBase     : 0x00007ff6`09c50000
   +0x100 LoadTime         : _LARGE_INTEGER 0x01d9a0eb`ed2ec4a2
   +0x108 BaseNameHashValue : 0x4c900b25
   +0x10c LoadReason       : 4 ( LoadReasonDynamicLoad )
   +0x110 ImplicitPathOptions : 0
   +0x114 ReferenceCount   : 2
   +0x118 DependentLoadFlags : 0
   +0x11c SigningLevel     : 0 ''
```

Zaten genelde de ilk load olan modül image'ın kendisi olur. Buradaki önemli alanları aşağıda açıklamaya çalıştım (aslında tüm alanlar önemli ama hepsini açıklamaya kalkarsam yazı bitmez..)

**~~InLoadOrderLinks:~~** 

**~~InMemoryOrderLinks:~~** 

**~~InInitializationOrderLinks:~~** 

**DllBase:** Process hafizasina yüklenen dll'in memory'deki adresini gösterir.

**FullDllName:** Process hafızasına yüklenen dll'in fullpath'ini tutar.

**BaseDllName:** Process hafızasına yüklenen dll'in ismini tutar.

Bir sonraki entry'e bakmak için flink alanı içerisindeki değeri bulmalıyız:

```cpp
lkd> dx -id 0,0,ffffcd0685106080 -r1 ((ntkrnlmp!_LIST_ENTRY *)0x18e0e8b32a0)
((ntkrnlmp!_LIST_ENTRY *)0x18e0e8b32a0)                 : 0x18e0e8b32a0 [Type: _LIST_ENTRY *]
    [+0x000] Flink            : 0x18e0e8b3110 [Type: _LIST_ENTRY *]
    [+0x008] Blink            : 0x7fffefb9c4d0 [Type: _LIST_ENTRY *]
```

beliritlen adres 0x18e0e8b31 olarak gösteriliyor. Bu adresi de _LDR_DATA_TABLE_ENTRY yapısı ile görüntülediğimizde:

```cpp
lkd> dt nt!_LDR_DATA_TABLE_ENTRY 0x18e0e8b3110 -y based
   +0x058 BaseDllName : _UNICODE_STRING "ntdll.dll"
```

notepad.exe'den sonra yüklenen modülün ismi ise ntdll.dll gözüküyor. Buradaki soru şu; *ben her modülü görüntülemek için tek tek böyle uğraşacak mıyım??* tabii ki de hayır. Windbg _LIST_ENTRY yapısını görüntüleyebilmemiz için bizlere birkaç komut sunar. Örneğin process'e yüklenen modülleri şu şekilde görüntüleyebiliriz:

```cpp
lkd> dl 0x0000018e`0e8b3110
0000018e`0e8b3110  0000018e`0e8b3760 0000018e`0e8b32a0
0000018e`0e8b3120  0000018e`0e8b3770 0000018e`0e8b32b0
0000018e`0e8b3760  0000018e`0e8b3d70 0000018e`0e8b3110
0000018e`0e8b3770  0000018e`0e8b3d80 0000018e`0e8b3120
0000018e`0e8b3d70  0000018e`0e8b5210 0000018e`0e8b3760
0000018e`0e8b3d80  0000018e`0e8b5220 0000018e`0e8b3770
0000018e`0e8b5210  0000018e`0e8b5530 0000018e`0e8b3d70
0000018e`0e8b5220  0000018e`0e8b5540 0000018e`0e8b3d80
0000018e`0e8b5530  0000018e`0e8b5850 0000018e`0e8b5210
0000018e`0e8b5540  0000018e`0e8b5860 0000018e`0e8b5220
0000018e`0e8b5850  0000018e`0e8b5cd0 0000018e`0e8b5530
0000018e`0e8b5860  0000018e`0e8b5ce0 0000018e`0e8b5540
0000018e`0e8b5cd0  0000018e`0e8b60e0 0000018e`0e8b5850
```

Diğer yöntem ise !list komutunu kullanmaktır.

```cpp
lkd> !list -x "dt nt!_LDR_DATA_TABLE_ENTRY -y fulldll" 0000018e`0e8b32a0
   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\system32\notepad.exe"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\ntdll.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\KERNEL32.DLL"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\KERNELBASE.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\GDI32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\win32u.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\gdi32full.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\msvcp_win.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\ucrtbase.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\USER32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\combase.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\RPCRT4.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\shcore.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\msvcrt.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\WinSxS\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.19041.1110_none_60b5254171f9507e\COMCTL32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\IMM32.DLL"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\bcryptPrimitives.dll"

   +0x048 FullDllName : _UNICODE_STRING "--- memory read error at address 0x0000018e`0e8c6ac0 ---"

   +0x048 FullDllName : _UNICODE_STRING "--- memory read error at address 0x0000018e`0e8c6700 ---"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\kernel.appcore.dll"

   +0x048 FullDllName : _UNICODE_STRING "--- memory read error at address 0x0000018e`0e8c6980 ---"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\clbcatq.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\MrmCoreR.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\SHELL32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\windows.storage.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\system32\Wldp.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\shlwapi.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\MSCTF.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\OLEAUT32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\system32\TextShaping.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\efswrt.dll"

   +0x048 FullDllName : _UNICODE_STRING "--- memory read error at address 0x0000018e`0e8d5bc0 ---"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\wintypes.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\twinapi.appcore.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\oleacc.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\textinputframework.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\CoreUIComponents.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\CoreMessaging.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\System32\WS2_32.dll"

   +0x048 FullDllName : _UNICODE_STRING "C:\Windows\SYSTEM32\ntmarta.dll"

   +0x048 FullDllName : _UNICODE_STRING ""
```

> Bu komutlar ile ilgili detaylı bilgiyi buradan bulabilirsiniz: [Debugger Commands - Windows drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-commands)

Veya direkt olarak !peb komutuyla hem ldr sorting field'ları hem de modülleri listeleyebiliriz:

```cpp
lkd> !peb 0x00000004`d1af4000
PEB at 00000004d1af4000    
    Ldr                       00007fffefb9c4c0
    Ldr.Initialized:          Yes
    Ldr.InInitializationOrderModuleList: 0000018e0e8b3130 . 0000018e0e8ebbe0
    Ldr.InLoadOrderModuleList:           0000018e0e8b32a0 . 0000018e0e8ed250
    Ldr.InMemoryOrderModuleList:         0000018e0e8b32b0 . 0000018e0e8ed260
                    Base TimeStamp                     Module
            7ff609c50000 bdd4adcd Dec 03 14:32:29 2070 C:\Windows\system32\notepad.exe
            7fffefa30000 6349a4f2 Oct 14 21:05:38 2022 C:\Windows\SYSTEM32\ntdll.dll
            7fffee490000 068524ca Jun 20 04:50:02 1973 C:\Windows\System32\KERNEL32.DLL
            7fffed160000 e1ac3f79 Dec 23 10:40:41 2089 C:\Windows\System32\KERNELBASE.dll
            7fffeeb10000 eeb3a47d Nov 26 09:41:01 2096 C:\Windows\System32\GDI32.dll
            7fffed610000 0dcd0213 May 03 23:26:59 1977 C:\Windows\System32\win32u.dll
            7fffed460000 b89e115a Feb 25 06:41:14 2068 C:\Windows\System32\gdi32full.dll
            7fffed980000 39255ccf May 19 18:25:03 2000 C:\Windows\System32\msvcp_win.dll
            7fffed6f0000 2bd748bf Apr 23 04:39:11 1993 C:\Windows\System32\ucrtbase.dll
            7fffee2f0000 32a2a2e9 Dec 02 12:35:37 1996 C:\Windows\System32\USER32.dll
            7fffee600000 03e7e147 Jan 29 13:15:35 1972 C:\Windows\System32\combase.dll
            7fffef8c0000 2261afdc Apr 12 08:49:16 1988 C:\Windows\System32\RPCRT4.dll
            7fffeed20000 29534f79 Dec 21 17:28:09 1991 C:\Windows\System32\shcore.dll
            7fffedf00000 564f9f39 Nov 21 01:31:21 2015 C:\Windows\System32\msvcrt.dll
            7fffd1460000 db2b08ef Jul 09 08:23:59 2086 C:\Windows\WinSxS\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.19041.1110_none_60b5254171f9507e\COMCTL32.dll
            7fffee080000 68ff10be Oct 27 09:27:10 2025 C:\Windows\System32\IMM32.DLL
            7fffed580000 856685b0 Dec 02 22:17:04 2040 C:\Windows\System32\bcryptPrimitives.dll
            7fffef530000 6869db26 Jul 06 05:10:46 2025 < Name not readable >
            7fffeec80000 9370b239 May 21 04:13:29 2048 < Name not readable >
            7fffeafc0000 f0713fcd Oct 30 09:42:21 2097 C:\Windows\SYSTEM32\kernel.appcore.dll
            7fffeaa50000 06bc4541 Aug 01 00:23:13 1973 < Name not readable >
            7fffee550000 a7c9263e Mar 15 21:13:18 2059 C:\Windows\System32\clbcatq.dll
            7fffdc860000 0b3246d4 Dec 15 05:58:28 1975 C:\Windows\System32\MrmCoreR.dll
            7fffeedd0000 e7fc7f4e May 02 09:35:58 2093 C:\Windows\System32\SHELL32.dll
            7fffeb1c0000 8eecb4fc Dec 26 08:05:00 2045 C:\Windows\SYSTEM32\windows.storage.dll
            7fffecb60000 db45726f Jul 29 09:13:03 2086 C:\Windows\system32\Wldp.dll
            7fffeec20000 19bb5737 Sep 06 17:52:39 1983 C:\Windows\System32\shlwapi.dll
            7fffef780000 0e8d3a56 Sep 26 18:42:14 1977 C:\Windows\System32\MSCTF.dll
            7fffeeb50000 61567b6b Oct 01 06:07:23 2021 C:\Windows\System32\OLEAUT32.dll
            7fffd02c0000 63a36c45 Dec 21 23:27:49 2022 C:\Windows\system32\TextShaping.dll
            7fffba0f0000 97acfd33 Aug 21 15:10:27 2050 C:\Windows\System32\efswrt.dll
            7fffc9850000 0d302819 Jan 05 00:03:21 1977 < Name not readable >
            7fffe9010000 55e08c48 Aug 28 19:28:56 2015 C:\Windows\SYSTEM32\wintypes.dll
            7fffe5c10000 60d2769c Jun 23 02:47:40 2021 C:\Windows\System32\twinapi.appcore.dll
            7fffe7090000 24cdd509 Jul 26 18:13:13 1989 C:\Windows\System32\oleacc.dll
            7fffdc610000 b13cfbc7 Mar 24 08:57:27 2064 C:\Windows\SYSTEM32\textinputframework.dll
            7fffe9ff0000 ce358de3 Aug 18 23:30:27 2079 C:\Windows\System32\CoreUIComponents.dll
            7fffea6d0000 f1ac3d92 Jun 26 07:56:50 2098 C:\Windows\System32\CoreMessaging.dll
            7fffee9c0000 aff3315b Jul 18 05:18:03 2063 C:\Windows\System32\WS2_32.dll
            7fffec290000 3d60ad04 Aug 19 11:32:04 2002 C:\Windows\SYSTEM32\ntmarta.dll
```

Bu arada not düşeyim; windows internal serisi sonunda (thread ve memory sonraki post konuları olacak sanırım) malware sample'larını analizini yapacağız ve analiz tekniklerinden bahsetmeye çalışacam. Ayrıca istek bir post veya herhangi bir sorunuz olursa bana mail, linkedin üzerinden ulaşabilirsiniz.

Takip etmeyi bırakmayın, sağlıcakla kalın :)

# Kaynakça ve Referanslar

1. Windows Internals Seventh Edition Part 1 (2017), Microsoft
2. Veriyapıları ve Algoritmalar, Toros Rıfat Çölkesen
3. [Windows Process Internals: Memory Forensics [Part 2] – ldrmodules \| By Kirtar Oza - eForensics](https://eforensicsmag.com/windows-process-internals-a-few-concepts-to-know-before-jumping-on-memory-forensics-part-2-ldrmodules-by-kirtar-oza/)
4. 
