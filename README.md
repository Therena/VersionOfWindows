# Read version record from Windows

Demonstrate the different ways of reading the version information from Windows. <br/>

Getting the Windows version sound in first glance easier as it is. It depends on many configuration setting and also on how the application was build
which version exact number you will get.

<a name="VersionRead"></a>
## Version read from the operating system

![Read Windows Version](https://github.com/Therena/VersionOfWindows/blob/master/Images/ReadWindowsVersion.png?raw=true)

## Table of contents

- [Version read from the operating system](#VersionRead)
- [Characteristic of the APIs](#Characteristic)
- [API descriptions](#APIDescriptions)
	- [API GetVersion](#APIGetVersion)
	- [API GetVersionEx](#APIGetVersionEx)
	- [API Kernel32Library](#APIKernel32Library)
	- [API RegistryCurrentVersion](#APIRegistryCurrentVersion)	
	- [API RegistryCurrentVersionNumbers](#APIRegistryCurrentVersionNumbers)
	- [API RtlGetNtVersionNumbers](#APIRtlGetNtVersionNumbers)
	- [API RtlGetVersion](#APIRtlGetVersion)
	- [API VersionHelper](#APIVersionHelper)
- [Windows Compatibility Mode](#WindowsCompatibilityMode)
- [Compatibility Manifest](#CompatibilityManifest)
- [License](#License)

<a name="Characteristic"></a>
## Characteristic of the APIs

| API                           | Accuracy  | Deprecated | Documented  | Compatibility Mode | Compatibility Manifest |
|-------------------------------|-----------|------------|-------------|--------------------|------------------------|
| GetVersion                    | yes       | yes        | User Mode   | dependent          | yes                    |
| GetVersionEx                  | yes       | yes        | User Mode   | dependent          | yes                    |
| Kernel32Library               | no        | no         | no          | independent        | no                     |
| RegistryCurrentVersion        | no        | no         | no          | independent        | no                     |
| RegistryCurrentVersionNumbers | unknown   | no         | no          | independent        | no                     |
| RtlGetNtVersionNumbers        | unknown   | no         | no          | independent        | no                     |
| RtlGetVersion                 | yes       | no         | Kernel Mode | dependent          | no                     |
| VersionHelper                 | yes       | no         | User Mode   | independent        | yes                    |

<a name="APIDescriptions"></a>
## API descriptions

<a name="APIGetVersion"></a>
### API GetVersion

Microsoft documentation: [GetVersion on MSDN](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getversion)

```cpp
#include <Windows.h>
const auto version = GetVersion();
```

The "GetVersion" WinAPI function returns the verion in one DWORD. The parts of the version can be extracted using the "LOWORD" and "LOWORD" macros.
The function is deprecated since Windows 8.1 and requires an Compatibility Manifest added to the application (See below). Furthermore it depends
on the Windows Compatibility Mode settings for the application which Windows Version will get returned (See section below).

<a name="APIGetVersionEx"></a>
### API GetVersionEx

Microsoft documentation: [GetVersionEx on MSDN](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getversionexw)

```cpp
#include <Windows.h>
OSVERSIONINFOEX versionInformation{};
SecureZeroMemory(&versionInformation, sizeof(OSVERSIONINFOEX));
versionInformation.dwOSVersionInfoSize = sizeof(OSVERSIONINFOEX);
GetVersionEx(reinterpret_cast<OSVERSIONINFO*>(&versionInformation));
```

The "GetVersionEx" WinAPI function returns the Windows version using an struct as out parameter. The parts of the version are stored as member variables in the struct.
The function is deprecated since Windows 8.1 and requires an Compatibility Manifest added to the application (See below). Furthermore it depends
on the Windows Compatibility Mode settings for the application which Windows Version will get returned (See section below).

<a name="APIKernel32Library"></a>
### API Kernel32Library

```cpp
#include <Windows.h>
#pragma comment(lib, "version")
const auto bufferSize = GetFileVersionInfoSize(L"kernel32.dll", nullptr);

const auto versionInformationBuffer = std::vector<byte>(bufferSize, 0);
const auto bufferFilledSize = GetFileVersionInfo(L"kernel32.dll", 0, bufferSize, versionInformationBuffer.data());

UINT versionLength = 0;
const auto queryValueResult = VerQueryValue(versionInformationBuffer.data(), 
    L"\\", 
    reinterpret_cast<LPVOID*>(&m_Version), 
    &versionLength);
```
Here the Windows version is determined by getting the file version of the "kernel32.dll".This by it natural independent of any compatibility manifest or compatibility mode.
Mostly that works well for the major and minor version of Windows, but for the build version part it delivers often not an accurate results.

<a name="APIRegistryCurrentVersion"></a>
### API RegistryCurrentVersion

```cpp
#include <Windows.h>
const auto result = RegOpenKeyExW(HKEY_LOCAL_MACHINE,
    LR"(SOFTWARE\Microsoft\Windows NT\CurrentVersion)", 0, KEY_READ, &currentVersionKey);

uint32_t dataLength = 256;
std::vector<byte> buffer(dataLength, L'\0');
const auto queryResult = RegQueryValueEx(currentVersionKey,
    L"CurrentVersion", nullptr, nullptr, buffer.data(), reinterpret_cast<LPDWORD>(&dataLength));
```

This solution is getting the Windows version directly from the Registry. There is an "CurrentVersion" value in the registry path
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```
Unfortunately it seems to doesn't get updated anymore since the Windows version 6.3.
This solution is independend of any compatibility manifest or compatibility mode.

<a name="APIRegistryCurrentVersionNumbers"></a>
### API RegistryCurrentVersionNumbers

```cpp
#include <Windows.h>
const auto result = RegOpenKeyExW(HKEY_LOCAL_MACHINE,
    LR"(SOFTWARE\Microsoft\Windows NT\CurrentVersion)", 0, KEY_READ, &currentVersionKey);

uint32_t dataLength = 256;
std::vector<byte> buffer(dataLength, L'\0');
auto queryResult = RegQueryValueEx(currentVersionKey,
    L"CurrentMajorVersionNumber", nullptr, nullptr, buffer.data(), reinterpret_cast<LPDWORD>(&dataLength));
queryResult = RegQueryValueEx(currentVersionKey,
    L"CurrentMinorVersionNumber", nullptr, nullptr, buffer.data(), reinterpret_cast<LPDWORD>(&dataLength));
queryResult = RegQueryValueEx(currentVersionKey,
    L"CurrentBuildNumber", nullptr, nullptr, buffer.data(), reinterpret_cast<LPDWORD>(&dataLength));
```

This solution is getting the Windows version directly from the Registry as well. There is the Windows version defined in these values:
- CurrentMajorVersionNumber
- CurrentMinorVersionNumber
- CurrentBuildNumber

The the following registry path:
```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion
```
The version number seems to be always accurat but as these registry values are not defined in any documentation this cannot be guatenteed.
This solution is independend of any compatibility manifest or compatibility mode.

<a name="APIRtlGetNtVersionNumbers"></a>
### API RtlGetNtVersionNumbers

```cpp
#include <Windows.h>
typedef VOID(NTAPI* RtlGetNtVersionNumbersFunc)(LPDWORD pdwMajorVersion, LPDWORD pdwMinorVersion, LPDWORD pdwBuildNumber);
const auto ntDll = GetModuleHandle(L"ntdll.dll");
const auto rtlGetNtVersionNumbers = (RtlGetNtVersionNumbersFunc)GetProcAddress(m_NtDll, "RtlGetNtVersionNumbers");

rtlGetNtVersionNumbers(reinterpret_cast<LPDWORD>(&mmajorVersion), 
    reinterpret_cast<LPDWORD>(&minorVersion), 
    reinterpret_cast<LPDWORD>(&buildNumber));
```

This API also seems to give an accurate version number which is independend of compatibility manifest and compatibility mode.
Unfortunately it is completely undocumented for use in kernel and user mode.

<a name="APIRtlGetVersion"></a>
### API RtlGetVersion

Microsoft documentation: [RtlGetVersion on MSDN](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-rtlgetversion)

```cpp
#include <Windows.h>
const auto ntDll = GetModuleHandle(L"ntdll.dll");
const auto rtlVersion = (RtlGetVersionFunc)GetProcAddress(ntDll, "RtlGetVersion");
const auto result = m_RtlVersion(&versionInformation);
```

The "RtlGetVersion" is documented for kernel mode but not for user mode code. It is able to provide an accurate version number but it is dependend on 
the compatibility mode (See section below) in case it is used in an user mode application.

<a name="APIVersionHelper"></a>
### API VersionHelper

Microsoft documentation: [VersionHelper on MSDN](https://learn.microsoft.com/en-us/windows/win32/sysinfo/version-helper-apis)

```cpp
#include <VersionHelpers.h>
const auto isWindows10 = ::IsWindows10OrGreater();
```

This is an API which can help to determine the versions of Windows like Windows 10. It isn't able to provide the exact number of the version.
In case the goal is only to differntiate Windows system that API works very accurate and is also the currently from Microsoft recommended API.
The version helper API is independend of the compatibility manifest and compatibility mode.

<a name="WindowsCompatibilityMode"></a>
## Windows Compatibility Mode

Microsoft documentation: [Make older apps or programs compatible with Windows](https://support.microsoft.com/en-us/windows/make-older-apps-or-programs-compatible-with-windows-783d6dd7-b439-bdb0-0490-54eea0f45938)

Windows has the option to run applications in an lower Windows version. For example it can be defined that an application running on Windows 10 should run in a Windows 7 compatibility environment.
This causes that some of the WinAPI will return the Windows 7 version number even if the application runs on Windows 10.

https://user-images.githubusercontent.com/18515874/212569524-4e96e82d-5e70-4a9e-a49d-7789fbcbc35e.mp4

<a name="CompatibilityManifest"></a>
## Compatibility Manifest

Microsoft documentation: [Comaptibility manifest on MSDN](https://learn.microsoft.com/en-us/windows/win32/sysinfo/targeting-your-application-at-windows-8-1)

The compatibility manifest defines with which Windows version the application is compatible with. The manifest will let Windows provide a different set of options to the running application.
The Windows version provided by some WinAPI are also different dependent on the compatibility manifest. For example in case the compatibility manifest doesn't
contain the Windows 10, the manifest dependend APIs will provide the Windows 8.1 version number even if the application will run on Windows 10.

```xml
<?xml version="1.0" encoding="utf-8"?>
<assembly manifestVersion="1.0" xmlns="urn:schemas-microsoft-com:asm.v1">
    <assemblyIdentity version="1.0.0.0" name="VersionOfWindows.app"/>
    <!-- 
        This manifest entries are important for getting the Windows version correctly
        while using the Version Helper header
    -->
    <compatibility xmlns="urn:schemas-microsoft-com:compatibility.v1">
        <application>
            <!-- A list of the Windows versions that this application has been tested on
           and is designed to work with. Uncomment the appropriate elements
           and Windows will automatically select the most compatible environment. -->

            <!-- Windows Vista -->
            <supportedOS Id="{e2011457-1546-43c5-a5fe-008deee3d3f0}" />

            <!-- Windows 7 -->
            <supportedOS Id="{35138b9a-5d96-4fbd-8e2d-a2440225f93a}" />

            <!-- Windows 8 -->
            <supportedOS Id="{4a2f28e3-53b9-4441-ba9c-d69d4a4a6e38}" />

            <!-- Windows 8.1 -->
            <supportedOS Id="{1f676c76-80e1-4239-95bb-83d0f6d0da78}" />

            <!-- Windows 10 -->
            <supportedOS Id="{8e0f7a12-bfb3-4fe8-b9a5-48fd50a15a9a}" />
        </application>
    </compatibility>
</assembly>
```

<a name="License"></a>
## License

[MIT License](https://github.com/Therena/VersionOfWindows/blob/master/LICENSE.txt)
