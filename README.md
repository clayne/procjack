# ProcJack

ProcJack is a code injection toolkit with which you can inject your code into a running process.  Injection is achieved by the same technique as [ReflectiveDLLInjection](https://github.com/stephenfewer/ReflectiveDLLInjection) does, i.e. the combination of `CreateRemoteThread` and `VirtualAllocEx`.

The key difference of ProcJack from other injection toolkits are:
- You can invoke a DLL-exported function in the target process (not DllMain)
- You can pass a string into your injected code through the command line arguments
- Injection footprint can be completely cleaned up i.e. unloading the injected module and freeing the allocated pages

The supported architecture of a target process is Native x86/x64 and WoW64.  No ARM support.  No Linux support.

## How to build

Open "Native Tools Command Prompt" of Visual Studio and run `NMAKE` under the root of this repository.

The following binaries will be generated under bin/[x86|amd64]:

- pj.exe: Injector program
- spy.dll: Sample DLL to inject

Other binaries such as expeb.exe or expeb[32|64].bin are for development purpose only.

## How to play

Let's inject the sample spy.dll!

1. Identify PID of a target process.  In this example, let's use notepad.exe below: PID=12884.

```
G:\10_Git\procjack>tasklist | find "notepad"
notepad.exe                  12884 Console                    1     11,996 K
```

2. Because the sample function of spy.dll prints a debug message via `OutputDebugString`, you need to attach your favorite debugger to the target process to see a visible result.

3. Run the following command.  The command says "Load spy.dll, and invoke a function of ordinal number 100 in the process of PID 12884".  The string "Hello!" is an optional parameter.  If you pass multiple (space-delimited) strings from the command line, you can get them as a null-delimited string in the function to be invoked (See the next section).

```
> G:\10_Git\procjack> bin\amd64\pj.exe 12884 bin\amd64\spy.dll?100 Hello!
Hijacking: PID=3254 (WIN64) TID=25d8 VM=0000028CC90D0000
```

VM=0000028CC90D0000 in the output above means the injector remotely allocated a region at 0000028CC90D0000 in the target in order to store code and data.  This address would be helpful when you need to debug the target process for some reason.

4. On the debugger console attached to the target, you'll see an output like this.  Injection done!

```
ModLoad: 00007ffe`54780000 00007ffe`54805000   G:\10_Git\procjack\bin\amd64\spy.dll
Injection Package:
 Page = 0000028CC90D0000
 DLL  = G:\10_Git\procjack\bin\amd64\spy.dll
 PEB  = 0000009D2E3D6000
 Args = Hello!
```

5. After the invoked function is finished, everything is cleaned up.  You can make sure spy.load is unloaded and the address 0000028CC90D0000 is no longer available.

```
0:001>  lm m spy
start             end                 module name
0:001> db 0000028CC90D0000 l10
0000028c`c90d0000  ?? ?? ?? ?? ?? ?? ?? ??-?? ?? ?? ?? ?? ?? ?? ??  ????????????????
```

## How to create your own DLL to inject

The injector pj.exe invokes an exported function by specifying an ordinal number in a target process.  There is no special restriction.  What you need to do is to write a DLL and to export a function with an ordinal number.

Exporting Functions from a DLL by Ordinal Rather Than by Name | Microsoft Docs<br />
https://docs.microsoft.com/en-us/cpp/build/exporting-functions-from-a-dll-by-ordinal-rather-than-by-name

The function can receive one pointer pointing to meta information, that is represented as the structure `Package` defined as follows.  For example, you can access the optional arguments through `Package::args`.

```cpp
struct Package {
  uint8_t shellcode[SHELLCODE_CAPACITY];
  wchar_t dllpath[260];
  HybridPointer<PEB> peb;
  HybridPointer<void> kernel32;
  HybridPointer<void*(WINAPI)(void*)> xxxLoadLibrary;
  HybridPointer<uint32_t(WINAPI)(void*)> xxxFreeLibrary;
  HybridPointer<void*(WINAPI)(void*, void*)> xxxGetProcAddress;
  char args[1];
};
```
