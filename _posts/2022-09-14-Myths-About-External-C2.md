---
layout: post
title: "Myths About External C2"
featured-img: Myths-About-External-C2/background
categories: [MalwareDev, C/C++]
---

# Introduction

In this blog post I will show you how to build a External C2 in your C2. Excuse me I can not show any actual code in my C2 Framework (Falcon One).


# Table of Content

1. What is the External C2?
	1. Is it useful?
2. Understand named pipe
	1. What is the Named Pipe?
	2. Types of Pipes
	3. How it works?
3. Build a Basic Demo
	1. Demo Structure
    2. Teamserver
    3. Third-Party Server
    4. Named Pipe communication
        1. Named Pipe Server
        2. Named Pipe Client
    5. How is the Agent looks like?
        1. Create Named pipe
        2. Write Function
        3. Read Function
    6. Third-Party Client
        1. Sockets
        2. Send Socket Function
        3. Receive Socket Fucntion
        4. Open Named pipe handle
        5. Pipe Write/Read Function
    7. Execution

# What is the External C2?

Cobalt Strike 3.6 introduced a new feature that's called External C2, to provide the operator a power to build his own communication channel.  

I will go through why it's powerful feature, but before that I would let you imagen how is the communication should be.   

![1](/assets/img/posts/Myths-About-External-C2/1.png)

As you see in the above flow, in the Attacker Machine we found the teamserver, and Third-Party Server we called it "Connector", The teamserver should provide a local port connection (In our case its TCP sockets), to let the third-party server connects with it to take the commands from the teamserver and send it to the custom channel which is Dropbox in our case, then when the commands get executed and the results on the Dropbox then the third-party server will take these results and send it back to the teamserver.  

And in the Target Machine we can found Third-Party client and the Main Agent. In the normal case the thrid-party should locate a memory space to inject the shellcode of the agent into it and execute the agent, but in our case I will run the agent manully. when the agent start running it'll create a named pipe to send and receive the data with the third-party client. and the main goal of the third-party is take the new tasks (commands) from the Dropbox and send it to the agent process through named pipe and receive the task results and send it back to the dropbox.

## Is it useful? 

I will let you answer this question by yourself.  
Lets assume there is company uses/trust dropbox for example. and allows the in and out connection.    
Looking for some words of wisdom? it is unrealistic to use untrusted domain to get your shell connection. So, in this case you can use dropbox to initial your connection and start communicating via trusted domain. So, is it useful or not?

# Understand Windows pipes

To be aware there is alot of abuses you can do in named pipe I may go through them in the up coming posts. but now lets talk about the mechanism of named pipes.   
Named pipe is documented by Microsoft, [Full documentation of Named Pipe](https://docs.microsoft.com/en-us/windows/win32/ipc/pipes)  

## What is the Windwos Pipe?

Windows Pipe helps when you need to communicate with two applications / processes using shared memory.   
You can interact with this shared memory like file object, like ReadFile() and WriteFile().
Named Pipe is implemented on **First in First out (FiFo)** which means when you write data to a pipe and needs to read it, once you read it the data will not be available anymore, the data will be poped out.   
But what if we have three application communicates togther and need to keep the data available. In this case you can use WinAPI function that's called `PeekNamedPipe` in windows.
I will go through it soon, but shortly this API can be used to read the data without removing.  

## Types of Windows Pipes

There is two types of windows pipes.

1. Named Pipes. 
2. Anonymous pipes.

Named Pipes have a name, but anonymous pipes doesn't have name.

For Example, Named piped like `\\.\Pipe\JustExample` 


## How it works?

Pipes is implemented based on a server and client, which the server create a named pipe then the client communicates through it.  
Named pipes has two methods of communication. `half-duplex` and `duplex`.  
Half-duplex will let the client side write data to the server with no repsond.  
In Duplex the client side can write data to the server and the server write to the client.  

