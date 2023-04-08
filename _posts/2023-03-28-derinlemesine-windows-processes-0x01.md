---
layout: post
title: Derinlemesine Windows Processes (0x01)
date: 2023-03-28 17:27 +0300
categories: [Derinlemesine Windows Serisi, Processes]
tags: [windows, internal, process, kernel-mode, user-mode]
---

![banner.gif](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/banner.gif)

Windows işletim sisteminin tepeden tırnağa objelerden oluştuğunu *Derinlemesine Windows Objects ve Handles (0x01)* yazısında belirtmiştik. Eğer objeler hakkındaki yazıları okumadıysanız ve objeler hakkında bilgi sahibi değilseniz, önce Windows Object serisini bitirmenizi şiddetle tavsiye ederim. Şimdi process'lerle devam edelim.

# Nedir Bu Process?

Normalde bilgisayar programları disk'te saklanır ve sizin onları kullanmanızı beklerler. Siz bir programı açtığınız zaman ise programın kodları belleğe aktarılır ve bir process haline gelirler. Kısa tanımıyla process'ler diskteki pasif program (çalışmayan bir program) kodlarının belleğe aktarılarak aktif duruma geçmesi denilebilir. O halde programları diskteki ve memory'deki hali olarak iki kısma ayırabiliriz. 

Şimdi gelin bir process'in oluşum sürecini beraber inceleyelim.

# Process'in Doğuşu

Windows sistemlerde process oluşumunu sağlayan Windows API fonksiyonları vardır. Bu fonksiyonların en temel örnekleri `CreateProcess` ve `CreateProcessAsUser` fonksiyonlarıdır. 

> Process oluşturmak için bitek bu fonksiyonlara muhtaç değiliz. Windows API fonksiyonlarını incelerseniz bir çok process oluşturmaya yarayan fonksiyonlar görebilir/öğrenebilirsiniz.

Tabii windows yazarları windows'u karmaşıklaştırmak için ellerinden geleni yaptıklarından olay bir tek CreateProcess ile bitmiyor. CreateProcess fonksiyonu call edildiği zaman oluşan function call flow aşağıdaki gibidir:

![process_creation_flow_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/process_creation_flow_01.png)

Bir uygulamayı run edildiğinde bu uygulamanın kullandığı tüm DLL’ler process memory’e yüklenir. Her Process en az bir tane thread'e sahip olma zorunluluğu olduğu için main thread'de oluşur. 

> Program kodlarını çalıştıran process değil, thread’in kendisidir.

Aşağıda CreateProcess API'si çağrıldıktan sonra yürütülecek adımlar aşağıda kısaca açıklanmıştır:

1- Thread bu CreateProcess() API'sinin process'e yüklenen image'ın IAT'si sayesinde hangi DLL'de ve hangi memory adresinde olduğunu bilmektedir.

2- Thread bu adresi belirledikten sonra CreateProcessInternal() ile devam edecektir.

3- Daha sonra kernel ile user modu arasındaki köprünün kurulacağı kısım için, CreateProcessInternal() fonksiyonu ntdll.dll’deki NtCreateUserProcess() (Gerçek NtCreateUserProcess() API’si değildir, kerneldeki fonksiyonun bir yansımasıdır) API'sini SYSCALL (x86 sistemde SYSENTER) ederek çağırır ve artık kernel kısmında işlem sağlanmaya hazırdır. 

Burada anlatılanların tamamını görsel olarak aşağıdaki diyagramda görebilirsiniz:

![function_call_flow_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/function_call_flow_01.png)

4- Şimdi ise thread kernel modda herhangi bir kısıtlama olmadan çalışmaktadır. Komut'un devamında kernel modunda System Service Descriptor Table (SSDT) sayesinde user modunda syscall edilen NtCreateUserProcess()'in gerçek kodları bulunur. 

5- NtCreateUserProcess(), ntoskrnl.exe’nin system service routine’inden çağrıldığında bu istek Process fonksiyonu olduğu için Process manager’a yönlendirilmektedir.

![ssdt_flow_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/ssdt_flow_02.png)

6- Son olarak process manager ise gelen isteği sistem içerisinde işler. Komutların decamı process manager'ı ilgilendirir.

> SSDT table'ı görmek isteyenler No Virus Thanks'in geliştirdiği SSDT Table Viewer aracını indirip bu tablodaki fonksiyonları görebilirler.

![ssdt_table_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/ssdt_table_01.png)

Sol sütunda gördüğünüz Index numaraları ntdll'den syscall edilen fonksiyonun ssdt table'da bulunmasını sağlar. Bu numara genelde EAX değerine MOV edilerek SYSCALL yapılır. Örneğin aşağıda  NtCreateUserProcess fonksiyonu disassemble edilerek SSDT tablosundaki index numarası görülebilir:

![ntcreateuserprocess_syscall_number_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/ntcreateuserprocess_syscall_number_01.png)

0xC8 değerinin decimal karşılığı da tahmin edeceğiniz üzere 200'dür.

Process oluşumuna örnek vermek gerekirse:

```cpp
#include <Windows.h>

int main()
{
    LPCWSTR pAppName = L"C:\\Windows\\System32\\calc.exe";
    SECURITY_ATTRIBUTES pSecAtt;
    SECURITY_ATTRIBUTES tSecAtt;
    STARTUPINFO pSi;
    PROCESS_INFORMATION pPi;

    ZeroMemory(&pSecAtt, sizeof(pSecAtt));
    ZeroMemory(&tSecAtt, sizeof(tSecAtt));
    ZeroMemory(&pSi, sizeof(pSi));
    ZeroMemory(&pPi, sizeof(pPi));

    CreateProcess(pAppName, NULL, &pSecAtt, &tSecAtt, false, NORMAL_PRIORITY_CLASS, NULL, NULL, &pSi, &pPi);
}
```

Yukarıdaki C++ kodu CreateProcess komutu kullanılarak windows'un hesap makinesi uygulamasını çalıştırır. Demin anlattıklarımızın doğruluğundan emin olmak adına CreateProcess'ten sonra oluşan function call flow'u trace edelim:

![create_process_call_flow_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/create_process_call_flow_01.png)

Gördüğünüz gibi NtCreateUserProcess fonksiyonu ile syscall edildi ve user tarafındaki son adım işlendi. Ayrıca verdiğimiz parametreler ise api tracer tarafından şu şekilde görüntüleniyor:

![create_process_parameters_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/create_process_parameters_01.png)

# Virtual Memory

Windows, memory manager'ı ile her yeni oluşan process için özel bir memory alanı açar. Bu alan bilgisayarın harddisk kısmında oluşur ve harddisk boyutuyla sınırlıdır. Process belleği ise virtual memory'in bir parçasıdır ve RAM'de (yani physical memory) oluşur. Bu alan RAM belleğin boyutuyla sınırlıdır. Runtime sırasında memory manager, virtual memory'in bir kısmını fiziksel adrese (RAM'e) yazar. Burada gerçek veriler bulunur. Sistem bunu pagefile/swap kullanarak aktif olmayan process'lerin fiziksel RAM'i gereksiz yere işgal etmesini engellemek için yapar.

> Page kavramı virtual memory'in en küçük parçasına verilen addır. Default değeri 4kb yani 4096 byte'tır. Eğer bir uygulamanın kullanmadığı bellek geri alınacaksa ya da fiziksel RAM'den çıkartılıp diske yazılacaksa 4Kb'lık parçalar halinde yapılmak zorundadır. Buna da paging/swapping denir.

Aşağıda iki process'in virtual memory'den RAM'e yazılışını, ve bazı process bölümlerinin de disk'e yazılış mantığını görebilirsiniz:

![virtual_memory_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/virtual_memory_01.png)

Ayrıca Virtual Memory process memory (işlem alanı veya kullanıcı alanı) ve kernel memory (çekirdek alanı veya sistem alanı) olarak ikiye ayrılır ve sanal bellek, adres uzayının boyutuna bağlıdır. 

Örneğin 32 bit olan bir sistemi düşünürseniz adresleme 2^32 olmak zorunda olduğu için process ve kernel memory için toplam virtual adress space maksimum 4 GB'dir. Sistem bu alanı yarıya böler ve bir yarısını kernel için, diğer yarısını da process belleği veya kullanıcı alanı için ayırmıştır. Ayrılan alanın 0x00000000 ile 0x7FFFFFFF kısmı kullanıcı işlemleri için, 0x80000000 ile 0xFFFFFFFF arası ise çekirdek alanı veya sistem alanı için ayrılmıştır.

## 32-Bit ve 64-Bit Mimarilerde VM

32-bit sistemlerde her işlem için 2 GB işlem belleği ve toplamda 4 GB sanal adres alanı mevcuttur. Windows bellek yöneticisi, fazla sanal adresi çözmek için bir kısmını diske sayfalayarak serbest bırakır. Ayrıca, çekirdek belleği çoğunlukla ortaktır ve tüm işlemler tarafından paylaşılır. Bu mimaride, kullanıcı ve çekirdek arasında 64 KB'lık bir erişilemez boşluk bulunmaktadır. x64 mimarisi ise, çok daha büyük adres alanı sunar ve kullanıcı alanı 0x0000000000000000 - 0x000007ffffffffff aralığına, çekirdek alanı ise 0xffff080000000000 ve üzerine yayılır. Aradaki büyük adres boşluğu kullanılamaz. Kanonik olmayan bir adres kullanmaya çalışmak, sayfa hatası istisnasına neden olur.

Aşağıda 32 ve 64 bitlik VM adres dağılımı yapısını görebilirsiniz:

![x86_64_virtual_memory_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/x86_64_virtual_memory_01.png)

> Sanal adres aralığı her process için aynı olsa da (x00000000 - 0x7FFFFFFF), hem donanım hem de Windows, bu aralığa eşlenen fiziksel adreslerin her işlem için farklı olduğundan emin olur. Örneğin, iki işlem aynı sanal adrese eriştiğinde, her işlem fiziksel bellekte farklı bir adrese erişir. İşletim sistemi, her işlem için özel adres alanı sağlayarak, işlemlerin birbirinin verilerini üzerine yazmasını engeller. Sanal bellek alanı her zaman 2 GB yarılara bölünmek zorunda değildir; bu sadece varsayılan yapılandırmadır. Örneğin, aşağıdaki komutu kullanarak 3 GB boot switch'ini etkinleştirebilirsiniz. Bu komut, işlem belleğini 0x00000000 - 0xBFFFFFFF aralığından 3 GB'a yükseltirken, kernel belleği kalan 1 GB'lık alanı 0xC0000000 - 0xFFFFFFFF aralığında alır: 
> `bcdedit /set increaseuserva 3072`

## Process ve Kernel Memory Öğeleri

Process memory kullanıcı uygulamaları tarafından kullanılan bellektir. Process belleği, process'lerin aynı kernel alanını paylaştığını da unutmayın.

Kernel memory ise işletim sistemi ve cihaz sürücülerini içerir. Aşağıdaki ekran görüntüsü user alanı ve kernel alanı bileşenlerini göstermektedir.

![kernel_memory_contents_01.png](/assets/img/2023-03-28-derinlemesine-windows-processes-0x01/kernel_memory_contents_01.png)

Yukarıdaki diyagramdaki öğeleri kısa ve hızlıca açıklamak istersek:

**DLLs:** Bir process oluşturulduğunda, tüm ilişkili DLL'leri process memory'e yüklenir. Bu bölge, bir process'lee ilişkili tüm DLL'leri temsil eder.

**PEB:** Her process’in kendi PEB alanı vardır. Process’in adı, pid, DLL'ler ve process’in kendisi dahil olmak üzere belleğe yüklenen tüm PE(Portable Executable) dosyalarının bilgilerini depolar.

**Executable:** Yazının ilk başında da belirtildiği gibi diskteki pasif bir programın çift tıklanılarak process'e dönüştüğü programdır. Diğer ismi de Image olarak nitelendirilir.

**Executive (ntoskrnl.exe):** Windows işletim sisteminin kernel görüntüsü olarak bilinir. Tüm fonksiyonların gerçek kodları bu dosyanın altında yatar. Örneğin user tarafında NTDLL.dll dosyasındaki fonksiyonlar sayesinde syscall edebildiğimiz fonksiyonların asıl kodları bu dosyanın altında yatar. Ayrıca Memory Manager, I/O Manager, Object Manager, Process/Thread Manager gibi işletim sistemi bileşenlerini de kontrol eder. 

**Win32k.sys:** Kernel modunda olan bu sürücü, output cihazlarında (örneğin monitörlerde) grafikleri oluşturmak için kullanılan UI ve GDI hizmetlerini sağlar ayrıca GUI uygulamaları için işlevler ortaya çıkarır. Kısa tanımıyla grafik'lerle uğraşır diyin :)

**Hal.dll:** Kernel'deki cihaz sürücüleri donanımla doğrudan iletişim kurmak yerine hal.dll tarafından sağlanan fonksiyonları call ederek donanımla etkileşime geçer. Birnevi bilgisayar donanımları arasındaki köprüdür.

Bu yazının konusu process'ler olduğu için detaylı bir şekilde görülmeyecek fakat ilerleyen blog yazılarında bunlarla ilgili yazılar yazmak gibi bir düşüncem var. Takipte kalın :)

# ~~Process Yönetimi~~

## ~~CreateProcess()~~

## ~~CreateProcess|AsUser(), WithLogon(), WithToken()~~

## ~~OpenProcess()~~

## ~~TerminateProcess()~~

## ~~ExitProcess()~~

## ~~Get*/Set* Fonksiyonları~~

# Environment Block

Her process’in bir dizi environment variable'ı ve bunların değerlerini içeren bir environment bloğu vardır. Child process defaultta parent process hangi kullanıcının oturumu açıldıysa, onun environment variable’larını alır. Ayrıca CreateProcess veya CreateProcessAsUser fonksiyonu, child process’in farklı bir environment blok belirtmesini sağlayabilir. Bunun için process oluşturulurken lpEnvironment parametresine Environment Bloğun pointer’ı set edilir. 

Environment bloğu boş sonlandırılmış Unicode dizelerinden oluşan bir dizidir. Liste iki boş değerle (\0\0) sona ermelidir. Tüm environment variable blokları şu şekilde bir yapıya sahiptir:

ANSI:

```cpp
VAR_1=Value_1\0
VAR_2=Value_2\0
...
VAR_N=Value_N\0\0
```

UNICODE karakterler içeren environmet block'lar için de şu şekildedir:

```cpp
VAR_1=Value_1\0
VAR_2=Value_2\0
...
VAR_N=Value_N\0\0\0\0
```

## Environment Block Yönetimi

Bu bloğu yönetmek için belirli API'ler vardır. 

- `CreateEnvironmentBlock` belirli bir environment bloğunun kopyasını almak için kullanılabilir.

- `DestroyEnvironmentBlock` CreateEnvironmentBlock tarafından oluşturulan bir environment bloğunu serbest bırakmak için kullanılabilir.

- `GetEnvironmentStrings` fonksiyonu, çağıran process’e ait ortam değişkenlerini içeren environment bloğunun memory’deki pointer’ını return eder.

- `SetEnvironmentVariable` bir ortam değişkenini değiştirmek için kullanılır.

- `FreeEnvironmentStrings` işinizin bittiği environment bloğunu dolduran memory hücrelerini free eder.

Aşağıda örnek olarak process'in environment variable'ları getirilmiştir:

```cpp
#include <windows.h>
#include <tchar.h>
#include <stdio.h>
#include <iostream>

void main()
{
    LPTSTR lpszVariable = (LPTSTR)GetEnvironmentStrings();

    while (*lpszVariable)
    {
        std::cout << std::endl;
        wprintf(lpszVariable);
        lpszVariable += lstrlen(lpszVariable) + 1;
    }
}
```

Bir process, kendi child’ı olmayan başka bir sürecin ortam değişkenlerini asla doğrudan değiştiremez

# Process'lerde Inheritance

Child process, parent process’ten birkaç kaynak ve özellik alabilir. Bunu yapabilmek için process oluşturulurken ek parametreler girilmelidir.

## Handle Ineritance

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

Burada CreateFile API fonksiyonu kullanılarak “output.txt” adında bir dosya oluşturuluyor ve bu dosya işlemi, sa.bInheritHandle parametresini TRUE olarak belirterek miras alınacak şekilde oluşturuluyor. Daha sonra, STARTUPINFO yapıları belirtiliyor ve si.hStdOutput parametresi, dosya işlemi handle’ını standart çıkış olarak belirterek işleme miras olarak aktarılıyor. Son olarak, ReadFile ve WriteFile API fonksiyonları kullanılarak işlem çıktısı dosyaya yazılıyor ve bellekte boş yer kaplamamaları için handle’lar kapatılıyor.

## Current Directory Inheritance

Child process default olarak parent process’in geçerli dizinini inherit edebilir. CreateProcess fonksiyonu, parent process’in, child process için farklı bir geçerli dizin belirtmesini sağlar.

- Current Direcroty değiştirmek isteyen bir process, SetCurrentDirectory fonksiyonunu kullanabilir

- Process’in geçerli dizinini almak isteyen bir process, GetCurrentDirecrory fonksiyonunu kullanabilir.

## Environment Block Inheritance

Child process defaultta parent process hangi kullanıcının oturumu açıldıysa, onun environment variable’larını alır. CreateProcess veya CreateProcessAsUser fonksiyonu, child process’in farklı bir environment blok belirtmesini sağlayabilir. Bunun için process oluşturulurken lpEnvironment parametresine Environment Bloğun pointer’ı set edilir.

```cpp
#include <windows.h>
#include <tchar.h>
#include <stdio.h>
#include <strsafe.h>

#define BUFSIZE 4096

int _tmain()
{
    TCHAR chNewEnv[BUFSIZE];
    LPTSTR lpszCurrentVariable; 
    DWORD dwFlags=0;
    TCHAR szAppName[]=TEXT("ex3.exe");
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
    BOOL fSuccess; 

    // Copy environment strings into an environment block. 

    lpszCurrentVariable = (LPTSTR) chNewEnv;
    if (FAILED(StringCchCopy(lpszCurrentVariable, BUFSIZE, TEXT("MySetting=A"))))
    {
        printf("String copy failed\n"); 
        return FALSE;
    }

    lpszCurrentVariable += lstrlen(lpszCurrentVariable) + 1; 
    if (FAILED(StringCchCopy(lpszCurrentVariable, BUFSIZE, TEXT("MyVersion=2")))) 
    {
        printf("String copy failed\n"); 
        return FALSE;
    }

    // Terminate the block with a NULL byte. 

    lpszCurrentVariable += lstrlen(lpszCurrentVariable) + 1; 
    *lpszCurrentVariable = (TCHAR)0; 

    // Create the child process, specifying a new environment block. 

    SecureZeroMemory(&si, sizeof(STARTUPINFO));
    si.cb = sizeof(STARTUPINFO);

#ifdef UNICODE
    dwFlags = CREATE_UNICODE_ENVIRONMENT;
#endif

    fSuccess = CreateProcess(szAppName, NULL, NULL, NULL, TRUE, dwFlags,
        (LPVOID) chNewEnv,   // new environment block
        NULL, &si, &pi); 

    if (! fSuccess) 
    {
        printf("CreateProcess failed (%d)\n", GetLastError());
        return FALSE;
    }
    WaitForSingleObject(pi.hProcess, INFINITE);
    return TRUE;
}
```

# Process'lerde Duplication

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

# Kaynakça ve Referans

1. Windows Internals Seventh Edition Part 2 (2022), Microsoft
2. Windows 10 System Programming Part 1 (2020), Leanpub
3. Learning Malware Analysis, Packt
4. [Making NtCreateUserProcess Work - Hack.Learn.Share](https://captmeelo.com/redteam/maldev/2022/05/10/ntcreateuserprocess.html)
5. [Understand and manage Windows 10 virtual memory \| TechTarget](https://www.techtarget.com/searchenterprisedesktop/tip/Understand-and-manage-Windows-10-virtual-memory)
6. [Parent Process vs. Creator Process - Pavel Yosifovich](https://scorpiosoftware.net/2021/01/10/parent-process-vs-creator-process/)
7. [Profile: Ali Paşa Turhan \| Microsoft Learn](https://learn.microsoft.com/en-us/users/ali-pasa-turhan/collections/nk4xtpne804wo6)
8. [Derinlemesine Windows Objects ve Handles (0x01) \| 0x0ryctes](https://alipasaturhan.github.io/posts/derinlemesine-windows-objects-ve-handles-0x01/)
