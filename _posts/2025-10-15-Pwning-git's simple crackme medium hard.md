---
title: "Cracking Git's Medium-Hard Crack me :3"
date: 2025-08-28 16:43:03 +/-0200
author: B4shCr00k
categories: [crackmes]
tags: [crackme, reverse engineering]     # TAG names should always be lowercase
---

## Infos

Author: git

Language: C/C++

Difficulty: 2.3

Platform: Windows

Arch: x86-64

Binary Name : plzcrackme.exe

MD5: 49c66031be227cc5982daadfd7368e9d

SHA1: 0f01dfd5c1775dd7b605c992903d67bbafa3051f

SHA256: 67b06c9c003f0c26c319d82b1fc6436207eaf0e3ed31f438312be8349225272f

Link: [https://crackmes.one/crackme/68e2b4652d267f28f69b738e](https://crackmes.one/crackme/68e2b4652d267f28f69b738e)

![The Function Name](/assets/img/1-1.png)


## initial testing 
![The Function Name](/assets/img/1-2.png)

the binary seems to be simple asks for a key and waits for input if the input is incorrect it prints "incorrect"

i tried inputing a long string to see if i get a segfault but nothing so this probably won't be a buffer overflow also inputing an empty string doesn't work so these critical cases are well handled 



## Static Analysis 

First of all the binary doesn't seem to be packed so il fire up ida and take a look at the main function of the binary, i immediately noticed some base64 encrypted strings i see more than one so maybe they will get decrypted and concatenated to get the password.

![The Function Name](/assets/img/strings.png)

next we already saw that the program prints "incorrect" if the password is wrong so il look for the string and its refs so i quickly get a look at how the check happens.

well i found some strings that tend to look like a password :D 


![The Function Name](/assets/img/1-3.png)

after testing all of them none worked ):

![The Function Name](/assets/img/1-4.png)

well even our main plan didn't work searching for the string "incorrect" didn't return any resutls which is weird so ig it is encrypted and we already saw some encrypted strings so yea we r getting closer (i hope m not wasting my time).

![The Function Name](/assets/img/1-5.png)

i found this snippet inside the main function that looks like it will be used for decryption mainly because of the alphabet string that might be used to decrypt the base64 strings 

well that was easy all we gotta do is move to dynamic analysis put a breakpoint on this function and single step until somewhere we will see the decrypted password in the memory or some register 

![The Function Name](/assets/img/1-6.png)
![The Function Name](/assets/img/1-7.png)

Agh ! ofc it won't be this easy 

classic anti debugging techniques so we first have to bypass them and get to the actual function we care about 

enough of static analysis

## Dynamic Analysis 

attaching to the binary then entering the password will immedietaly shut down both processes as we saw before ```isdebuggerpresent()``` is being called so what il do is set a breakpoint right before the call so we can bypass it somehow 

the instruction at offset 16C6 is 3 instructions away from the ```isdebuggerpresent()``` call so il set a breakpoint there 

to do so we just get the exe base and add it to the offset 

![The Function Name](/assets/img/1-8.png)

nice we stopped just before the call to is debugger present, we can do this the hard way by going through the call but when we get to the test to see if the debugger is present or not which happens in ```eax``` where the return value in windows x64 calling convention gets stored, for example here it tests if its zero or not then jumps to a function if its not zero which means if a debugger is attached it will jump to that function which after going through we notice it calls some ints i can guess used for process termination so what we can do is simply after the call set eax to 1 and this way we bypass the check :D this is the right way to avoid breaking the program 

but after reading more instructions down i notice more calls to similar functions to see if a debugger is present so what il do since m lazy il patch one of the calls to just jump over all these calls to where i think the actual juice is 

i set a jump instruction to the rva 17AF which is 

```
imul rax,r9,64
```

i simply chose this because i didn't see any more calls to sus functions that most likely are anti debuggin techniques 

BEFORE: 

![The Function Name](/assets/img/1-9.png)

AFTER: 

![The Function Name](/assets/img/1-10.png)

i kept single stepping through the instructions and easily i got this 

![The Function Name](/assets/img/1-11.png)

well that was easy ^^

the password is : TotallySecureKey123

ironic 

## Conclusion 

the crackme is definitely not a medium to hard its easy to medium but it was fun  
















