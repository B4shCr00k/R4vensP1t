---
title: "Create Your First Windows Driver"
date: 2025-07-28 4:08:03 +/-0200
author: B4shCr00k
categories: [Documentations,poc]
tags: [windows,kernel,driver,guide]     # TAG names should always be lowercase

---



# Hello From Kernel

In this guide i will help you write your first windows kernel driver using kmdf 

# This Guide Will Cover

 - What are drivers

 - Prerequisite 

 - User Mode VS Kernel Mode 

 - Code WalkThrough

 - Compiling and loading the driver 


## What Are Drivers 

A driver is a piece of software that allows the operating system to communicate with hardware devices. In our case, we are developing what's known as a software driver, a type of driver that facilitates communication between user mode and kernel mode, without necessarily interacting with physical hardware. These drivers are essential for tasks that require privileged access to system resources, such as monitoring, hooking, or custom control mechanisms.

When it comes to malware and rootkits, drivers take on a more malicious role. In this context, a driver can be used to gain low-level access to the operating system, often bypassing security mechanisms like User Account Control (UAC) or antivirus protections. Kernel-mode drivers used by rootkits can hide files, processes, or registry keys, intercept system calls, and exert stealthy control over the system—making them one of the most powerful tools in advanced malware development.

## Prerequisite 

 * First of all you really don't want to test drivers on your main (host) machine since any fatal mistake will make your system crash therefore having a testing environnement is so important i advice using a virtual machine you can use vmware (free version) or virtual box which is free as well

 * once you have the testing machine ready disable memory integrity options from the windows security settings then we need to enable test signing to do so run the following commands on an elevated cmd (run as administrator)

```c
bcdedit.exe -set loadoptions DISABLE_INTEGRITY_CHECKS
bcdedit.exe -set TESTSIGNING ON
```
you might get something like "The value is protected by Secure Boot policy and cannot be modified or deleted" in this case you have secure boot on you should go google how to disable secure boot for your machine usually from the bios 

 * now we are done from setting up the testing machine lets set up the machine where we will be coding our diver, first you will need to install visual studio, sdk and wdk [here](https://learn.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk) is a guide by microsoft 
 
 * next we can set up kernel debugging using windbg but its pretty complicated and unnecessary for now so we you can just download DebugView on the testing machine

 * finally we create a kmdf empty project using visual studio add a main.c under your source folder and delete the driver you will find under driver files since we dont need it 

## User Mode VS Kernel Mode 

for security, stability, and control reasons execution is split into two modes :

--- user mode where normal programs like browsers games text editors run it cannot access memory used by the os and needs to call the os using high level api windows functions in order to do such things, programs in user mode has a private virtual memory meaning when a process runs it thinks it owns the entire memory to it self this is implemented for security reasons as well as other reasons read [this](https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/virtual-address-spaces) for more infos on virtual memory 

--- kernel mode where the kernel itself, drivers and os programs run it can literally do anything in this system in kernel mode you have full access to hardware, memory and system resources all programs that run in kernel mode share the same virtual space meaning if something happens to one program it will crash the entire system leading to a BSOD or other fatal errors 

drivers are like a bridge that allow us to run code in kernel space 

## Code WalkThrough

Now We will code a simple hello world from the kernel poc 

we need to be very careful when coding drivers and we need to follow a spesific structure open main.c and lets start coding

 - first we include  `#include <ntddk.h>` which has everything we need to code our driver 

 - next we define something called a tag which is used to track down memory allocations made by the driver we use this for debugging so we can easily know what driver allocated this memory space `#define DRIVER_TAG 'gaT'` first we write the name in reverse due to the little endian and use single quotes we also need to define a  `UNICODE_STRING` which is a structure for unicode strings which a lot of low level functions need it has 3 fields `length` which is the size of the buffer in bytes and not null terminated `MaximumLength` which is the max size the buffer can hold `buffer` the actual unicode string so `UNICODE_STRING g_RegPath;`

 - next we define our entrypoint which is `NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)` the driverobject is a struct that holds all the important infos about our drivers it has things like `DriverObject->DriverUnload` which is the code that will get executed once the driver is unloaded this struct is very important and is like a handle to control the driver, the registery path is a unicode string that has something like "\Registry\Machine\System\CurrentControlSet\Services\MyDriver" we will print it later

 - inside our function we will be using something called`DbgPrint();` which is just like `printf()` in user mode this function will allow us to debug the driver using print statements we can later read using debugging tools, next we print the driverobject and the registerypath like this 

 ```c
 	DbgPrint("[+] Driver Object: %p\n", DriverObject);
	DbgPrint("[+] Registry Path: %p\n", RegistryPath);
 ```

 - next we will allocate space for our buffer to populate our `g_RegPath` unicode string struct we do this so can save the reg path for later use since its only save to access it i the DriverEntry only to do this we will get into memory allocation in kernel mode, there are more than one function to allocate space in kernel mode unlike use mode where we can use malloc we can use `ExAllocatePool2()` which is a new function used to allocate space in memory safely so `g_RegPath.Buffer = (PWSTR)ExAllocatePool2(POOL_FLAG_PAGED, RegistryPath->Length, DRIVER_TAG)` we are allocating space for our buffer and we store the pointer inside `g_RegPath.Buffer` the first argument is `POOL_FLAG_PAGED` read more about memory pools [here](https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/virtual-addres) next we can how many bytes we will allocate `RegistryPath->Length`  which is the length of the regpath buffer, finally we have the tag we will use for this memory space `DRIVER_TAG` which is Test in our case next we run a test to see if it the buffer is still null just like with malloc

 ```c
 	if (g_RegPath.Buffer == NULL)
	{
		DbgPrint("[-] Failed To Allocate Space For The Reg");
		return STATUS_NO_MEMORY;
	}
 ```

 - then we copy the registerypath from the param into our buffer using memcpy `memcpy(g_RegPath.Buffer, RegistryPath->Buffer, RegistryPath->Length);` also we set the length to same length `	g_RegPath.Length = g_RegPath.MaximumLength = RegistryPath->Length;` and finally we dbgprint a message `	DbgPrint("Param Key Copy %wZ\n",g_RegPath)`

 - now we need to set `DriverObject->DriverUnload` into our unload function which will run when the driver unload `DriverObject->DriverUnload = UnloadMe` so make sure you defined this function before the DriverEntry Function so the compiler can recognize it 

 - now we code the actual UnloadMe function it takes one param which is the driverobject struct for now we will just free the allocated space which is something you should always do in kernel mode 

 ```c
 void UnloadMe(PDRIVER_OBJECT DriverObject) {

	UNREFERENCED_PARAMETER(DriverObject);
	DbgPrint("[+] Driver unloaded successfully.\n");
	DbgPrint("[+] Driver Object: %p\n", DriverObject);
	
	if (g_RegPath.Buffer != NULL) {
		ExFreePool(g_RegPath.Buffer);
		g_RegPath.Buffer = NULL;
	}
	
	DbgPrint("[+] Driver Unloaded\n");
}
 ```

 we used `ExFreePool(g_RegPath.Buffer);` which is just like `free()` in um 

## Compiling and loading the driver

to build the project just build in visual studio and make sure you do so in debug mode or all the debug messages wont appear 

- to install the driver we will use `Service Controller` (sc.exe) to do so we run this command 
`sc create HelloWorldDriver type= kernel binPath= C:\MyDrivers\HelloWorldDriver.sys` change the path into ur dirver's path you can also change HelloWorldDriver into whatever name u want now open DebugView and enable all the kernel capture options under capture 

- finally we load the driver with `sc start HelloWorldDriver` or whatever name u chose you should see the messages appear in your DebugView interface if u dont its because you didnt enable the correct options under the capture tab to unload the driver we use `sc stop HelloWorldDriver`

FULL CODE : 

```c
#include <ntddk.h>

#define DRIVER_TAG 'bdwh' 

UNICODE_STRING g_RegPath;

void UnloadMe(PDRIVER_OBJECT);

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {

	DbgPrint("[+] Driver loaded successfully.\n");
	DbgPrint("[+] Driver Object: %p\n", DriverObject);
	DbgPrint("[+] Registry Path: %p\n", RegistryPath);

	g_RegPath.Buffer = (PWSTR)ExAllocatePool2(POOL_FLAG_PAGED, RegistryPath->Length, DRIVER_TAG);
	if (g_RegPath.Buffer == NULL)
	{
		DbgPrint("[-] Failed To Allocate Space For The Reg");
		return STATUS_NO_MEMORY;
	}
	
	memcpy(g_RegPath.Buffer, RegistryPath->Buffer, RegistryPath->Length);

	g_RegPath.Length = g_RegPath.MaximumLength = RegistryPath->Length;
	DbgPrint("Param Key Copy %wZ\n",g_RegPath);

	DriverObject->DriverUnload = UnloadMe;
	
	
	return STATUS_SUCCESS;
}

void UnloadMe(PDRIVER_OBJECT DriverObject) {

	UNREFERENCED_PARAMETER(DriverObject);
	DbgPrint("[+] Driver unloaded successfully.\n");
	DbgPrint("[+] Driver Object: %p\n", DriverObject);
	
	if (g_RegPath.Buffer != NULL) {
		ExFreePool(g_RegPath.Buffer);
		g_RegPath.Buffer = NULL;
	}
	
	DbgPrint("[+] Driver Unloaded\n");
}

```