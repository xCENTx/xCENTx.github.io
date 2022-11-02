---
title: Accessing PCSX2 Memory
category: Memory Management
date: 2022-11-2
encrypted_text: true
---

<p align="center">
<img src="https://forums.pcsx2.net/images/darktheme/logo_default.png">
</p>

# Topic Summary
> Program: PCSX2 v1.7+  
> Release Date: 2002
> Requirements:  
> - PCSX2 v1.7.2384 (or newer) with the game 
> - SOCOM 2 U.S Navy SEALs NTSC (patch r0001)

## Example PS2 RAW Format Cheat Code
```
// Apply Render Fix
2033CD68 100000DB

// Disable Render Fix
2033CD68 106000DB
```

## Example PNACH Cheat Code
// File Name: `0F6FC6CF.pnach` (0F6FC6CF is the games CRC code)  
// This is enabled at game launch (or whenever cheats are enabled)  
// These generally cannot be disabled without some prior knowledge (original value and joker commands)  
// Users tend to hotswap pnach files instead of doing it properly  
```
gametitle=SOCOM II - U.S. Navy SEALs (NTSC-U)
comment=Test File Generated by xCENTx
patch=1,EE,2033CD90,extended,00000000
```
PCSX2 provides native cheat support with the use of PNACH files. Being that the PS2 is 20+ years old at this point and that cheat software devices were common place during its lifecycle , the PlayStation 2 has a massive cheat library ready to be utilized in new ways thanks to emulation.  
In the past it was possible to access PCSX2 emulated game memory at the static location 0x20000000 , refer to the example raw cheat shown above, meaning it was very easy to utilize all of the cheats.  
With PCSX2 transitioning to x64 architecture it has become a bit harder to access PCSX2 memory as it's now required to make use of pointers.  

## Program Log  
PCSX2 provides a built in debug log that allows us to see exactly where they map some of the modules  
`Debug > Program Log. Under sources enable "Dev/Verbose Logging" and restart PCSX2`  

<p align="center">
<img src="https://i.imgur.com/T811ZUg.png">
</p>

EE Main Memory is what we are after. EE stands for EmotionEngine and is the CPU that handles all PS2 Memory Instructions.  
The entry point for this module will be exported to the log here everytime we launch PCSX2 but unfortunately thats not really good enough for trainer and ImGui Menu Purposes.  
```
Main Memory Manager              @ 0x00007FF700000000 -> 0x00007FF728000000 [640mb]
Mapping host memory for virtual systems...
	EE Main Memory                   @ 0x00007FF700000000 -> 0x00007FF702884000 [40mb]
	IOP Main Memory (2mb)            @ 0x00007FF704000000 -> 0x00007FF704211000 [2mb]
	VU0/1 on-chip memory             @ 0x00007FF708000000 -> 0x00007FF70800A000 [40kb]
Reserving memory for recompilers...
	Micro VU0 Recompiler Cache       @ 0x00007FF71C000000 -> 0x00007FF720000000 [64mb]
	Micro VU1 Recompiler Cache       @ 0x00007FF720000000 -> 0x00007FF724000000 [64mb]
	R5900-32 Recompiler Cache        @ 0x00007FF710000000 -> 0x00007FF714000000 [64mb]
	R3000A Recompiler Cache          @ 0x00007FF714000000 -> 0x00007FF716000000 [32mb]
	VIF0 Unpack Recompiler Cache     @ 0x00007FF716000000 -> 0x00007FF716800000 [8mb]
	VIF1 Unpack Recompiler Cache     @ 0x00007FF718000000 -> 0x00007FF718800000 [8mb]
```

## Reversing PCSX2 Source Code
We are looking for things directly related to "EE Main Memory" So, we will just go to PCSX2 git page and search ""EE Main Memory"
After some digging we will eventually find something like this and it's rather clear that this is how the baseAddress for EE Main Memory is set.
```
File= System.cpp
HostMemoryMap::EEmem   = base + HostMemoryMap::EEmemOffset;
```

Narrowing our search parameter to "EEMem" will give a grand total of 5 results. 
The result we are after is the DLL Export for `EEmem` (located in `System.cpp`)
```c++
#ifdef _WIN32
    _declspec(dllexport) uptr EEmem, IOPmem, VUmem, EErec, IOPrec, VIF0rec, VIF1rec, mVU0rec, mVU1rec, bumpAllocator;
#else
    __attribute__((visibility("default"), used)) uptr EEmem, IOPmem, VUmem, EErec, IOPrec, VIF0rec, VIF1rec, mVU0rec, mVU1rec, bumpAllocator;
#endif
```

## Obtaining Exported Module List and Base Address
Using either Cheat Engine or Process Hacker we can view a processes exported modules.
Process Hacker -> Modules -> PCSX2 -> Properties -> PCSX2 -> Exports.
Lets open up process hacker and peek modules. Go to PCSX2x64 and open it up. Go to the exports tab.
We can see EEmem is in the export table. 

<p align="center">
<img src="https://i.imgur.com/ozRpk7k.png">
</p>

This means that we can just type "PCSX2x64.EEmem" to get the base address for EEmem every single time in Cheat Engine
now we can obtain the base address for EEmem with each build of PCSX2 v1.7 without even having to change a thing.

<p align="center">
<img src="https://i.imgur.com/jF67Vyd.png">
</p>

## C++ Internal Trainer
The next thing we need to do is build our trainer. I personally prefer to be internal so for the purposes of this tutorial , I will provide source code for an Internal DLL project.
We need to get the module, module base address and the base address for EEmem. doing all of this internal is very easy.
```c++
// module
HMODULE module = GetModuleHandle(NULL);

// Base Address
uintptr_t moduleBaseAddress = (uintptr_t)module;

// EEmem Base
uintptr_t EEmem = *(uintptr_t*)GetProcAddress(module, "EEmem");     // Used to get the base address of the exported module EEmem
```

With all of that out of the way we can finally start making some cheats for PCSX2 again. What's great about PCSX2 v1.7 is that we no longer need to recompile virtual memory to apply patches when internal (I have another write up on this for previous versions of PCSX2). We also get to take advantage of a lot of the new features being implemented into the dev builds. 1.7 also offers more compatibility with PS2 games and overall seems to run a bit better in my experience

<p align="center">
<img src="https://i.imgur.com/xR0FwTL.png">
</p>

## Source Code
```c++
#include <windows.h>
#include <iostream>

bool INFO = FALSE;

namespace PCSX2 {
    HMODULE module = NULL;
    uintptr_t BaseAddress = NULL;
    uintptr_t EEmem = NULL;
}

namespace Offsets {
    int RENDERFIX = 0x33CD68;   //  2033CD68
}

void displayOptions()
{
    system("cls");
    std::cout << R"(--------------------------------------
EXAMPLE GAME OPTIONS:
*SOCOM 2 NTSC MUST BE RUNNING
[1] GAME Info
[2] Patch Render Fix
[END] QUIT
--------------------------------------
)";
    INFO = TRUE;
}

DWORD WINAPI MainThread(HMODULE hModule)
{
    //  OPEN CONSOLE
    AllocConsole();
    FILE* f;
    freopen_s(&f, "CONOUT$", "w", stdout);

    //  ESTABLISH PROC INFO
    PCSX2::module = GetModuleHandleA(NULL);
    PCSX2::BaseAddress = (uintptr_t)PCSX2::module;
    PCSX2::EEmem = *(uintptr_t*)GetProcAddress(PCSX2::module, "EEmem");

    //  CONSOLE DEBUG
    printf("PROCESS INFORMATION: \n");
    printf("PCSX2 Base Address: %llX\n", PCSX2::BaseAddress);
    printf("PCSX2 EEmem BaseAddress: %llX\n\n", PCSX2::EEmem);
    printf("[END] QUIT\n");
    printf("[INSERT] Example Options\n");

    bool dosomething = true;
    while (dosomething) {

        //  HOTKEY TO EXIT
        if (GetAsyncKeyState(VK_END) & 1) break;

        if (GetAsyncKeyState(VK_INSERT) & 1)
            if (!INFO)
                displayOptions();

        if (INFO) {

            //  EXAMPLE READ
            if (GetAsyncKeyState(VK_NUMPAD1) & 1) {
                uintptr_t RENDERaddr = (PCSX2::EEmem + Offsets::RENDERFIX);
                uintptr_t RENDERvalue = *(int*)(RENDERaddr);
                printf("--GAME INFORMATION\n");
                printf("Render Fix Address: %llX\n", RENDERaddr);
                printf("Render Fix Value: %X\n", RENDERvalue);
            }

            //  EXAMPLE WRITE
            if (GetAsyncKeyState(VK_NUMPAD2) & 1) {
                system("cls");
                uintptr_t RENDERaddr = (PCSX2::EEmem + Offsets::RENDERFIX);
                uintptr_t RENDERvalue = *(int*)(RENDERaddr);
                printf("PATCHING . . .\n");
                Sleep(1000);
                if (RENDERvalue != (int)0x100000DB)
                    *(int*)RENDERaddr = (int)0x100000DB;
                else {
                    printf("ALREADY PATCHED!");
                    Sleep(1500);
                }
                dosomething = false;
            }
        }
    }

    //  EXIT
    fclose(f);
    FreeConsole();
    FreeLibraryAndExitThread(hModule, 0);
    return 0;
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD  ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        CloseHandle(CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)MainThread, hModule, 0, nullptr));
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```