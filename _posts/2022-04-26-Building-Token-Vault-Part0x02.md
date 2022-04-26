---
layout: post
title: "Access Token Manipulation Part 0x02"
featured-img: Building-Token-Vault-Part0x01/background
categories: [MalwareDev, C/C++]
---

# Introduction

This is part 2 of Access Token Manipulation blog. In this blog post I'm gonna talk about building a Token Vault to store your stolen tokens in memory.

# Linked List

If you are building a C2 and you need to implement token vault into your C2 you may ask yourself what I need to store in the token vault?
For sure you will need to store the token, Process ID of that token, TokenIndex, Domain Name,and who is the owner of that process. that's it.
So, Now we know what we need to store into token vault. Let's answer how to store it? 

Actually you can do a Linked List that contains your data for example:

```CPP
typedef struct  _TOKEN_VAULT
{
    char User;
    char Domain;
    DWORD ProcessID;
    HANDLE Token;
    INT TokenIndex;
    struct _TOKEN_VAULT* Next;

}TokenVault, * PTokenVault;
```
Looks simple, right?

Q: Is this a new technique? 
A: No, many C2s are using that technique. Ref: [Metasploit Meterpreter Payload](https://github.com/rapid7/metasploit-payloads/blob/master/c/meterpreter/source/extensions/incognito/list_tokens.h#L7)


# Weaponization

If you remember in part 1 we built MakeToken, Stealtoken, and Revert2Self functions.
![1](/assets/img/posts/Building-Token-Vault-Part0x01/1.png)

So, in this blog, we are going to build three other functions first for adding a token, the second one for listing tokens, and the last one UseToken to reuse tokens in a token vault. So, Let's get a start in building our header file.
Simply we create a linked list to store our data into it.

![2](/assets/img/posts/Building-Token-Vault-Part0x01/2.png)

As we see in the above screenshot we have 5 members in this data structure. 

1. User
2. Domain
3. ProcessID
4. Token
5. TokenIndex

And lastly we have pointer that point to the next node.

Then we defined our Token Header Ptr. Why is it NULL? because if it NULL that means the Token vault is empty.

and lastly we have defined a prototype of our two functions.

## InsertToken

Let's go through InsertToken Function.

![3](/assets/img/posts/Building-Token-Vault-Part0x01/3.png)

InsertToken function takes 5 arguments as we discussed before. Then we have two other Pointers. One of these new pointers will be used as a new node to hold our data.

![4](/assets/img/posts/Building-Token-Vault-Part0x01/4.png)

Q: Why we assigned NULL to EntryPtr->Next ?
A: Because as we discussed before, that pointer will holder the location of the next pointer and as we adding new token that means that's the last node.

Then we need to check if the HeadPtr is NULL or not. If it NULL that means this is the first add in the token vault. So if it equal NULL we are going to pass the EntryPtr to it.

![5](/assets/img/posts/Building-Token-Vault-Part0x01/5.png)

Q: What if it not equal NULL? 
A: That means Token vault is not empty. So then we will need to find the last node. do you remember why we assigned NULL to EntryPtr? Becouse its the last node, right!
Exactly that what are we going to do now we are going to do a While true until we find the next pointer equal NULL.
Then we will modify that Next pointer to our EntryPtr.

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em;'>
<b>Note</b></p>
<p style='margin-left:1em;'>
Do not forget to set TokenIndex
</p></span>
</div>

![6](/assets/img/posts/Building-Token-Vault-Part0x01/6.png)

That's how we can create a Token Vault.

## ListToken

In the ListToken function we will need to create just one more pointer, that pointer will hold the HeaderPtr data and do a simple loop to extract the whole token.

![7](/assets/img/posts/Building-Token-Vault-Part0x01/7.png)

In the above screenshot, we just check if the HeaderPtr is empty or not.

If not we are going to that loop. 

![8](/assets/img/posts/Building-Token-Vault-Part0x01/8.png)

As you see at the end of the loop we move to another node by 

```cpp
CurrentNode = CurrentNode->Next;
```
## UseToken

So, now what if you need to use one of these stored tokens.
It's so simple you can do like we did in the ListToken function.
1. Check if the Token Vault isn't NULL.
2. Create a new Node.
3. Loop until you find that the given index equals the CurrentNode->TokenIndex.
4. Use ImpersonateLoggedOnUser.

![9](/assets/img/posts/Building-Token-Vault-Part0x01/9.png)


## Determine the owner of a process

How do determine the owner of a process to store the token in a token vault?

You can call [GetTokenInformation](https://docs.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-gettokeninformation?redirectedfrom=MSDN) function. by passing the Token Handle of that process. and for TokenInformationClass member you can pass `TokenOwner` to retrieve SID of the user. 
And to convert the SID to the actual user account you can use [LookupAccountSidA](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-lookupaccountsida?redirectedfrom=MSDN)

Then store that user account through 

```CPP
CurrentNode->Username = UserAccount;
```

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em;'>
<b>Note</b></p>
<p style='margin-left:1em;'>
Take care when we are going to check if the GetTokenInformation returned an error or not in the first call I mean when you NULL the TokenInformation member, It will return ERROR_INSUFFICIENT_BUFFER but that is the normal action when you NULL that member, so do not worry and keep going.
</p></span>
</div>

And here is how could you use these APIs.

![10](/assets/img/posts/Building-Token-Vault-Part0x01/10.png)


<hr>

If you have feedback please go ahead and DM me on Twitter, See you in the next blog post.


Peace out!✌️

<a href="https://www.buymeacoffee.com/ret2pwn" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: 41px !important;width: 174px !important;box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 3px 2px 0px rgba(190, 190, 190, 0.5) !important;" ></a>