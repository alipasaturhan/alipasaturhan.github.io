---
layout: post
title: Derinlemesine Windows API (0x01)
date: 2023-09-02 16:23 +0300
categories: [Derinlemesine Windows Serisi, Windows API]
tags: [windows, api]
---

# Windows DLL ve Header İlişkisi

Windows aslında arkaplanda çok düzenli bir mekanizmaya sahip bir işletim sistemidir. Bu düzen kullanıcının girdiği herhangi bir input'tan bilgisayarın output'una kadar geçerlidir.

Mesela yeni bir text dosyası oluşturmak istediğinizde, arkaplanda dosya oluşturmaya yarayan windows api fonksiyonu (CreateFile) call edilir. Daha sonra bu fonksiyon başka bir dll'e forward edilir ve syscall/sysenter yapılarak kernel diyarına geçiş yapılır. Gerisi ise kernel ile çalışan driver ve manager'lara kalmış işlemlerdir (burada yapılan işlemleri görebilmek için windbg kullanılabilir).

Peki biz dosya oluşturmayı sağlayan winapi fonksiyonunu direkt olarak call etsek nasıl olur? CreateFile fonksiyonu kernel32.dll modülüne ait bir fonksiyon olarak geçer. Bu fonksiyonu kullanmak istediğimizde ise dll'i değil, fonksiyonun tanımlandığı header'ı (Windows.h) include etmemiz gerekiyor.

Windows.h Windows API'sindeki tüm fonksiyonlar, programcılar tarafından kullanılan tüm yaygın makrolar ve Windows tarafından kullanılan veri türleri için bildirimler içeren header dosyasıdır.

![CreateFile_function_preview.png](/assets/img/2023-09-02-derinlemesine-windows-api-0x01/CreateFile_function_preview.png)

CreateFile fonksiyonunun kullanımına baktığımızda ise aslında filapi.h header'ında olduğunu görüyoruz. Demekki buradan şöyle bişey çıkarabiliriz:

![Dll_Headers_Connexion.png](/assets/img/2023-09-02-derinlemesine-windows-api-0x01/Dll_Headers_Connexion.png)

## Well Known DLLs

- kernel32.dll: Low level NTDLL wrappers.

- user32.dll: User interface primitives used by graphical programs with menus, toolboxes, prompts, windows ..

- shell.dll

- gdi32.dll: Basic drawing primitives.

- ole32.dll

- MSVCRT.DLL: Implementation of the C standard library stdlib.

- advapi.dll: Contains functions for system-related tasks such as registry and registry handling.

- WS_32.DLL: Winsock2 library contains a socket implementation.

- Ntdll.dll: Interface to Kernel. Not used by Windows programs directly.

- Wininet.dll: Provides high level network APIs, for instance, HttpOpenRequest, FtpGetFile …

# Main Headers

- windows.h: Basic header file of Windows API
- WinError.h: Error codes and strings
- tchar.h: Provides the macro _T(…) and TEXT(…) for Unicode/ANSI string encoding handling.
- wchar.h: Wide Character - UTF16 or wchar
- global.h
- ntfsb.h
- Winsock2.h: Network sockets
- Winbase.h: Windows types definitions
- WinUser.h: Windows Messages
- ShellAPI.h: Shell API
- ShFolder.h: Folder definitions
- Commdlg.h: Commom Controls (COM based)
- Dlgs.h: Dialog definitions
- IUnknown.h: COM header
- conio.h: Console Input/Output functions - it is heritage grom MSDOS.

# WinAPI Data Types

Windows API'lerle programlama yapmadan önce kendi syntax'ini ve veri tiplerini anlamakta fayda var. WinAPI C/C++ tabanında yazıldığına rağmen ben bunun ayrı bir dil olduğunu hayal ederim (sizin düşünmenize gerek yok). Çünkü windows kendi içinde çok fazla veri tipine sahip ve sadece kendi veri tiplerini kullanarak programlamaya izin veriyor. Örneğin integer veri tipini kabul etmez, dword veri tipini kabul eder. Halbuki ikisi de aynı şey (nerdeyse). Yazının ilerisinde ne demek istediğimi daha iyi anlayacaksınız.

## Function Prefix Types

İlk önce API fonksiyonlarının parametrelerinde, başlangıçlarında yazan iki-üç harflik ek'lere bakalım istiyorum. Örnek olarak CreateProcess fonksiyonunu inceleyelim:

```cpp
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

Parametre önlerinde (sağdakiler) duran lp, b, dw eklerini gördünüz mü? Bunlar parametrenin alabileceği veri tiplerini temsil eder. Ve dikkat ettiyseniz tanımlanan veri tipleriyle (ortadakiler) benzerlikleri var. Bunu gereksiz veya küçük bir bilgi gibi görebilirsiniz ama bu işlerde bilginiz pek yoksa en çok hata alacağınız kısım veri tipleri olacaktır.

Aşağıda tüm preprefix'leri ve anlamlarını beliten tabloyu görebilirsiniz:

| Prefix  | Type                 | Description                                | Variable Name Example              |
| ------- | -------------------- | ------------------------------------------ | ---------------------------------- |
| b       | BYTE or BOOL         | boolean                                    | BOOL bFlag; BOOL bIsOnFocus        |
| c       | char                 | Character - 1 byte                         | char cLetter                       |
| w       | WORD                 | word                                       |                                    |
| dw      | DWORD                | double word                                |                                    |
| i       | int                  | integer                                    | int iNumberOfNodes                 |
| u32     | unsigned [int]       | unsigned integer                           | unsigned u32Nodes                  |
| f or fp | float                | float point - single precision             | fInterestRate                      |
| d       | double               | float point - double precision             | dRateOfReturn                      |
| n       |                      | short int                                  |                                    |
| sz      | char* or const char* | Pointer to null terminated char array.     | char* szButtonLabel                |
| H       | HANDLE               | Handle                                     | HANDLE hModule; HMODULE hInstance; |
| p       | -                    | Pointer                                    | double* pdwMyPointer;              |
| lp      | -                    | Long Pointer                               | int* lpiPointer;                   |
| fn      | -                    | Function pointer                           |                                    |
| lpsz    |                      | Long Pointer                               |                                    |
| LP      |                      | Long Pointer                               |                                    |
| I       | -                    | Interface (C++ interface class)            | class IDrawable …                  |
| S       | -                    | Struct declaration                         | struct SContext { … }              |
| C       | -                    | Class declaration                          | class CUserData{ … }               |
| m_      | -                    | Private member variable name of some class | m_pszFileName                      |
| s_      | -                    | Static member of a class                   | static int s_iObjectCount          |

> Aklınızda bulunsun; windows api'lerin bu şekilde değişken isimlendirmesine macar notasyonu (hungarian notataion) deniyor.

# Genel Data Types

Windows API fonksiyonlara girdi olabilecek veri tipleri, return edilebilecek veri tipleri vs. tüm api fonksiyonları bu data tiplerini kullanır. 

| Data Type       | Definition                                  | Description                                                                      |
| --------------- | ------------------------------------------- | -------------------------------------------------------------------------------- |
| BOOL            | typedef int BOOL                            | Boolean variable true (non zero) or false (zero or 0)                            |
| BYTE            | typedef unsigned char BYTE                  | A byte, 8 bits.                                                                  |
| CCHAR           | typedef char CHAR                           | An 8-bit Windows (ANSI) character.                                               |
| DWORD           | typedef unsigned long DWORD                 | A 32-bit unsigned integer. The range is 0 through 4294967295 decimal.            |
| DWORDLONG       | typedef unsigned __int64 DWORDLONG          | 64 bits usigned int.                                                             |
| DWORD32         | typedef unsigned int DWORD32                | A 32-bit unsigned integer.                                                       |
| DWORD64         | typedef unsigned __int64 DWORD64            | A 64-bit unsigned integer.                                                       |
| FLOAT           | typedef float FLOAT                         | A floating-point variable.                                                       |
| INT8            | typedef signed char INT8                    | An 8-bit signed integer.                                                         |
| INT16           | typedef signed short INT16                  | A 16-bit signed integer.                                                         |
| INT32           | typedef signed int INT32                    | A 32-bit signed integer. The range is -2147483648 through 2147483647 decimal.    |
| INT64           | typedef signed __int64 INT64                | A 64-bit signed integer.                                                         |
| LPBOOL          | typedef BOOL far *LPBOOL;                   | A pointer to a BOOL.                                                             |
| LPBYTE          | typedef BYTE far *LPBYTE                    | A pointer to a BYTE.                                                             |
| LPCSTR, PCSTR   | typedef __nullterminated CONST CHAR *LPCSTR | pointer to a constant null-terminated string of 8-bit Windows (ANSI) characters. |
| LPCVOID         | typedef CONST void *LPCVOID;                | A pointer to a constant of any type.                                             |
| LPCWSTR, PCWSTR | typedef CONST WCHAR *LPCWSTR;               | A pointer to a constant null-terminated string of 16-bit Unicode characters.     |
| LPDWORD         | typedef DWORD *LPDWORD                      | A pointer to a DWORD.                                                            |
| LPSTR           | typedef CHAR *LPSTR;                        | A pointer to a null-terminated string of 8-bit Windows (ANSI) characters.        |
| LPTSTR          |                                             | An LPWSTR if UNICODE is defined, an LPSTR otherwise.                             |
| LPWSTR          | typedef WCHAR *LPWSTR;                      | A pointer to a null-terminated string of 16-bit Unicode characters.              |
| PCHAR           | typedef CHAR *PCHAR;                        | A pointer to a CHAR.                                                             |
| CHAR            | ANSI Char or char                           |                                                                                  |
| WCHAR           | Wide character 16 bits UTF16                |                                                                                  |
| TCHAR           | -                                           | A WCHAR if UNICODE is defined, a CHAR otherwise.                                 |
| UCHAR           | typedef unsigned char UCHAR;                | An unsigned CHAR.                                                                |
| WPARAM          | typedef UINT_PTR WPARAM;                    | A message parameter.                                                             |

Diğer önemli data tipleri aşağıdaki gibidir.

| Data Type | Description                                        |
| --------- | -------------------------------------------------- |
| HANDLE    | 32 bits integer used as a handle                   |
| HDC       | Handle to device context                           |
| HWND      | 32-bit unsigned integer used as handle to a window |
| LONG      |                                                    |
| LPARAM    |                                                    |
| LPSTR     |                                                    |
| LPVOID    | Generic pointer similar to void*                   |
| LRESULT   |                                                    |
| UINT      | Unsigned integer                                   |
| WCHAR     | 16-bit Unicode character or Wide-Character         |
| WPARAM    |                                                    |
| HINSTANCE |                                                    |

# Source Code Annotation Language (SAL)

Windows API'lerle çalışırken hangi parametrelerin değer döndürmek için kullanıldığını veya yalnızca girdi olarak kullanıldığını anlamak zor olabilir. Burada Windows'un kullandığı SAL yazım biçimi, hangi fonksiyon parametrelerinin input, readonly ve hangi parametrelerin output olduğunu _In\_ veya _out\_ gibi prefix'lerle bildirmeyi sağlar.

Aşağıda bu prefix'lerin ne anlam ifade ettiğini görebilirsiniz.

| SAL Annotatio | Description                                                                                                                         |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| _In\_         | Input parameter - read only argument no modified inside the by the function. Generally has the const qualifier such as const char*. |
| _In_Out_      | Optional input parameter, can be ignored by passing a null pointer.                                                                 |
| _Out\_        | Output paramenter - Argument is written by the called function. It is generally a pointer.                                          |
| _Out_opt_     | Optional output parameter. Can be ignored by setting it to null pointer.                                                            |
| _Inout\_      | Data is passed to the function and pontentially modified.                                                                           |
| _Outptr\_     | Output to caller. The value returned by written to the parameter is pointer.                                                        |
| _Outptr_opt_  | Optional output pointer to caller, can be ignored by passing NULL pointer.                                                          |

Yine CreateProcess fonksiyonu ile devam edersek örneği aşağıdaki gibidir:

```cpp
BOOL WINAPI CreateProcess(
  _In_opt_    LPCTSTR               lpApplicationName,      // const char*
  _Inout_opt_ LPTSTR                lpCommandLine,          // char*
  _In_opt_    LPSECURITY_ATTRIBUTES lpProcessAttributes,
  _In_opt_    LPSECURITY_ATTRIBUTES lpThreadAttributes,     // 
  _In_        BOOL                  bInheritHandles,
  _In_        DWORD                 dwCreationFlags,
  _In_opt_    LPVOID                lpEnvironment,
  _In_opt_    LPCTSTR               lpCurrentDirectory,
  _In_        LPSTARTUPINFO         lpStartupInfo,
  _Out_       LPPROCESS_INFORMATION lpProcessInformation
);
```

Eğer kendimiz de SAL formatında bir fonksiyon yazmak istersek sal.h header'ını include etmemiz gerekiyor. Çünkü standart C/C++ dili arasında bu yazım biçimi yoktur. Ayrıca bu yazım biçimi yalnızca MSVC - Microsoft Visual C++ compiler üzerinde çalışır.

> Eğer _opt\_ prefix'i varsa NULL olarak o parametreyi geçebilirsiniz, fakat bu prefix yoksa parametreye input vermek zorunludur.

SAL yazım biçiminin kullanarak programla örneğini aşağıda görebilirsiniz:

```cpp
#include <sal.h>     // Microsft's Source Code Annotation Language 
#include <iostream>

// Computes elementwise product of two vectors 
void vector_element_product(
      _In_ size_t size,
      _In_ const double xs[],
      _In_ const double ys[],
      _Out_      double zs[]
      ){
      for(int i = 0; i < size; i++){
          zs[i] = xs[i] * ys[i];
      }
}

void showArray(size_t size, double xs[]){
  std::cout << "(" << size << ")[ ";
  for(int i = 0; i < size; i++){
    std::cout << xs[i] << " ";
  }
  std::cout << "] ";
}

int main(){
  double xs [] = {4, 5, 6, 10};
  double ys [] = {4, 10, 5, 25};
  double zs [4];
  vector_element_product(4, xs, ys, zs);
  std::cout << "xs = "; showArray(4, xs); std::cout << "\n";
  std::cout << "ys = "; showArray(4, ys); std::cout << "\n";
  std::cout << "zs = "; showArray(4, zs); std::cout << "\n";

}
```

# ~~WinAPI'de Karakter Encoding~~

| Type    | Definition     |
| ------- | -------------- |
| LPSTR   | char*          |
| LPCSTR  | const char*    |
| LPWSTR  | wchar_t*       |
| LPCWSTR | const wchar_t* |
| LPTSTR  | TCHAR*         |
| LPCTSTR | const TCHAR*   |

# ~~Environment Variables~~

# Kaynakça ve Referanslar

1. [Windows API index - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/apiindex/windows-api-list)

2. [Windows Data Types (BaseTsd.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winprog/windows-data-types)

3. [Understanding SAL \| Microsoft Learn](https://learn.microsoft.com/en-us/cpp/code-quality/understanding-sal?view=msvc-170)
