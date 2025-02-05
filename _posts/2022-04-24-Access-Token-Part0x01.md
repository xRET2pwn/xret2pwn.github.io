---
layout: post
title: "Access Token Manipulation Part 0x01"
featured-img: Access-Token-Part0x01/background
categories: [MalwareDev, C/C++]
---



# Introduction

In this blog post, I'm going to show the most three common access token techniques.

1. Steal Token
2. Revert2Self
3. Make Token

In upcoming posts I'm gonna talk about how to build a token vault to store your tokens.


# Access Token Manipulation (Steal Token)

How many time did you use steal_token,  Revert2Self, and Make_token modules in C2 like Cobalt Strike? Too much, right?! 

In this section I will show you how to write your own Access Token Manipulation in C/C++.


## Short Forward

Token Manipulation is used to impersonate another running process token.

So basically we can do that through OpenProcess as **PROCESS_QUERY_LIMITED_INFORMATION**. 

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em;'>
<b>Note</b></p>
<p style='margin-left:1em;'>
There is no need to use <b>PROCESS_ALL_ACCESS</b>.</p>
<p style='margin-bottom:1em; margin-right:1em; text-align:right; font-family:Georgia'>
</p></span>
</div>



Then you can pass this Opened handle of the target process to OpenProcessToken to open a handle to the Access Token of the process. 

Then duplicate the token of the specified process through DuplicateTokenEx.

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em;'>
<b>Note</b></p>
<p style='margin-left:1em;'>
Here is no need to use <b>TOKEN_ALL_ACCESS</b>. You can just pass <b>MAXIMUM_ALLOWED</b> to the second member of DuplicatedTokenEx.
</p></span>
</div>

Then you can execute whatever you want by passing the stolen handle to CreateProcessWithTokenW.

### Weaponization

As we are trying to steal token of another process so we need to get that process id. So in the first few lines I just check if the user entered the process id of the target process or not.

Then initialize some variable to use them later.

![1](/assets/img/posts/Access-Token-Part0x01/1.png)


So, first mission we need to do is to open handle of the target process and that could be done through [OpenProcess](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)


```CPP
HANDLE OpenProcess(
  [in] DWORD dwDesiredAccess,
  [in] BOOL  bInheritHandle,
  [in] DWORD dwProcessId
);
```

OpenProcess takes 3 arguments. 

1. dwDesiredAccess // The access to the process object. Here is a list of Access Right you could use [Access Rights](https://docs.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights) but in our case we can use **PROCESS_ALL_ACCESS** or **PROCESS_QUERY_LIMITED_INFORMATION**, but It's recommended to use **PROCESS_QUERY_LIMITED_INFORMATION** for less suspicious activity.
2. bInheritHandle // This process inherit the handle
3. dwProcessId // Process Id of the target process

Then we need to check if the handle is opened successfully or not.

![2](/assets/img/posts/Access-Token-Part0x01/2.png)


After opening the target process handle, we will need to open handle of the process handle and that could be done through [OpenProcessToken](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocesstoken)

```cpp
BOOL OpenProcessToken(
  [in]  HANDLE  ProcessHandle,
  [in]  DWORD   DesiredAccess,
  [out] PHANDLE TokenHandle
);
```

OpenProcessToken takes 3 arguments.

1. ProcessHandle // the returned value of OpenProcess. 
2. DesiredAccess // Access Rights of the Token Handle.
3. TokenHandle // [OUT] Token Handle variable.

As always we need to check if the handle is opened successfully or not.

![3](/assets/img/posts/Access-Token-Part0x01/3.png)


Then we will need to duplicate that token handle and that could be done through [DuplicateTokenEx](https://docs.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-duplicatetokenex)

```cpp
BOOL DuplicateTokenEx(
  [in]           HANDLE                       hExistingToken,
  [in]           DWORD                        dwDesiredAccess,
  [in, optional] LPSECURITY_ATTRIBUTES        lpTokenAttributes,
  [in]           SECURITY_IMPERSONATION_LEVEL ImpersonationLevel,
  [in]           TOKEN_TYPE                   TokenType,
  [out]          PHANDLE                      phNewToken
);
```

DuplicateTokenEx takes 6 arguments

1. hExistingToken // the returned value of  OpenProcessToken 
2. dwDesiredAccess // Access Rights
3. lpTokenAttributes // Attributes of the Token
4. ImpersonationLevel // Impersonation Level of the token we are going to use [SECURITY_IMPERSONATION_LEVEL](https://docs.microsoft.com/en-us/windows/win32/api/winnt/ne-winnt-security_impersonation_level)
5. TokenType // Token Type
6. phNewToken // [OUT] New Token Handle variable.


![4](/assets/img/posts/Access-Token-Part0x01/4.png)


So, now we have the duplicated token handle, so now we have two options to create a new process through the duplicated token.

1. CreateProcessWithTokenW 
2. ImpersonateLoggedOnUser, and CreateProcessWithLogonW

**First Way**

[CreateProcessWithTokenW](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithtokenw) can be used to new process through the stolen token. 

CreateProcessWithTokenW takes 9 arguments

```cpp
BOOL CreateProcessWithTokenW(
  [in]                HANDLE                hToken,
  [in]                DWORD                 dwLogonFlags,
  [in, optional]      LPCWSTR               lpApplicationName,
  [in, out, optional] LPWSTR                lpCommandLine,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCWSTR               lpCurrentDirectory,
  [in]                LPSTARTUPINFOW        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

![5](/assets/img/posts/Access-Token-Part0x01/5.png)

**Second Way** 

[ImpersonateLoggedOnUser](https://docs.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-impersonateloggedonuser) can be used to impersonate the security context of a logged-on user through token. 

```cpp
BOOL ImpersonateLoggedOnUser(
  [in] HANDLE hToken
);
```

ImpersonateLoggedOnUser just takes the token which the duplicated token.

![6](/assets/img/posts/Access-Token-Part0x01/6.png)


![7](/assets/img/posts/Access-Token-Part0x01/7.png)


# Revert Impersonation Token (Revert2Self)

What if you need to return to your primary token, what do you need to do? 

There is an API that's called [RevertToSelf](https://docs.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-reverttoself) 

```cpp
BOOL RevertToSelf();
```

You just need to call that function with no arguments to revert the token and if the returned value is nonzero so you successfully revert the token. 
And we have done : ) looks easy, right?


# Make Token

As we are trying to make a new token for the user, we must have the credentials for that user.

![8](/assets/img/posts/Access-Token-Part0x01/8.png)

To log in with the user credentials and return the token handle, we can do that through [LogonUserA](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-logonusera) 

```cpp
BOOL LogonUserA(
  [in]           LPCSTR  lpszUsername,
  [in, optional] LPCSTR  lpszDomain,
  [in, optional] LPCSTR  lpszPassword,
  [in]           DWORD   dwLogonType,
  [in]           DWORD   dwLogonProvider,
  [out]          PHANDLE phToken
);
```

LogonUserA takes 6 arguments

1. lpszUsername // The username value. actually you can set UPN format but if you did that you must NULL the domain value.
2. lpszDomain // Domain Name.
3. lpszPassword // User's Password.
4. dwLogonType // The type of logon operation to perform
5. dwLogonProvider // Specifies the logon provider
6. phToken // [OUT] Token Handle variable.

![9](/assets/img/posts/Access-Token-Part0x01/9.png)

After returning the user token handle you can use ImpersonateLoggendOnUser as we did in the Steal Token section.


![10](/assets/img/posts/Access-Token-Part0x01/10.png)


You can use direct System calls to bypass any userland hooks. that could be done through [SysWhispers3](https://github.com/klezVirus/SysWhispers3) or you can Implement your own Syscall. and if you re-write this in CSharp you can use [DInvoke/DInvoke.DynamicInvoke at master . rasta-mouse/DInvoke](https://github.com/rasta-mouse/DInvoke/tree/master/DInvoke.DynamicInvoke)



<hr>

If you have feedback please go ahead and DM me on Twitter, See you in the next blogpost.


Peace out!✌️
  
<a href="https://www.buymeacoffee.com/ret2pwn" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: 41px !important;width: 174px !important;box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;" ></a>