---
title: "Remotely Resolving Imports"
date: 2025-07-28 16:08:03 +/-0200
author: B4shCr00k
categories: [Documentations]
tags: [windows,drivers,kernel]     # TAG names should always be lowercase

---

# what are imports 

basically imports are functions a pe needs to function properly, these functions are located inside dlls (dynamic link libraries) so first the executable needs to load the dll then find the functions he needs and get their addresses

now in order to know which functions an executable needs there is something called the IAT table which is a table inside the import directory in the pe structure, this table contains all the names of the functions the exe (or the dll) needs as well as their addresses

when the exe first lunches the iat will be empty just names but no addresses so windows automatically reads this table, finds which dlls contains the functions, loads them using a function called `LoadLibraryA()` which will patch this table with then use something like `GetProcAddress()` to get the function addresses then patch the iat table with the correct addresses so the exe now works well

# What is Manual Loading

instead of letting windows do all the stuff i just mentioned we do everything manually, Why? for a lot of reasons like windows uses high-level api functions like `LoadLibraryA()` or `GetProcAddress()` which in case the system has an edr installed will def be hooked therefore your malware will immedieatly gets detected to avoid this we manually patch the iat table 

# how is it done 

first of all since we are dealing with a pe you need to parse the pe and get the ntheader, this guide is about remotely resolving imports but in order to do so you would have to first write the pe into the target process with all the headers and sections 

so first of all 

- get the import directory rva (relative virtual address) which is stored in 
`ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress`
where IMAGE_DIRECTORY_ENTRY_IMPORT is a macro for 1 

- next we need the import descriptor `IMAGE_IMPORT_DESCRIPTOR` it is located at the virtual address so in order to get it 
```c
IMAGE_IMPORT_DESCRIPTOR* importDesc = (PIMAGE_IMPORT_DESCRIPTOR)(BaseAddress + ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);
```

- next we start looping through every dll we use the condition `while(importDesc->Name)` we first get the dll name which is stored in `importDesc->Name` so `const char* dllName = (char*)(BaseAddress + currentDesc->Name);` and then we load the library so we can get the functions inside it `HMODULE dllHandle = LoadLibraryA(dllName);` now we can access the dll what we need to do is read what fucntions the pe needs and get their addresses from the loaded dll 
- we have originalThunk and firstThunk 

originalThunk : where the pe stores the function names it needs whether using names or ordinal 
firstThunk : where the pe stores the actual addresses of these functions and this is what we need to patch 

- we can read them both this way 
```c
	PIMAGE_THUNK_DATA origThunk = (PIMAGE_THUNK_DATA)(BaseAddress + importDesc->OriginalFirstThunk);
	PIMAGE_THUNK_DATA firstThunk = (PIMAGE_THUNK_DATA)(BaseAddress + importDesc->FirstThunk);
```
now we have addresses to them we start by getting the function names inside the originalthunk `while(origThunk->u1.AddressOfData)` so there r two cases for functions whether we should import them by ordinal or by names lets start with ordinal 

- we do the following pick the `origThunk->u1.Ordinal` and (and &) it with `IMAGE_ORDINAL_FLAG64` which is 0x8000000000000000 in x64 what this does is check if the ordinal bit is set if true then we need to import by ordinal to do so we extract the ordinal by again using the & operator with the ordinal `WORD ordinal = (WORD)(origThunk->u1.Ordinal & 0xFFFF);` this way we extract the ordinal only finally we get the address `funcAddress = GetProcAddress(dllHandle, (LPCSTR)(uintptr_t)ordinal);`

- now for imports by name we need to first get the structure `PIMAGE_IMPORT_BY_NAME importByName = (PIMAGE_IMPORT_BY_NAME)(BaseAddress + origThunk->u1.AddressOfData);`
then we have the name `char* funcName = (char*)importByName->Name;` all we need to do is get the address with `funcAddress = GetProcAddress(dllHandle, funcName);`

- for the final step which we did all of this for we need to patch the iat table first we need its address
first we calculate the offset using `ULONGLONG offset = (BYTE*)firstThunk - (BYTE*)BaseAddress;` this gives us the offset of the entry inside the iat table we can get this from our process since we have the offset we just add it to the remote base `ULONGLONG remoteIATEntry = (ULONGLONG)RemoteAddress + offset;`  finally we path the iat table using `WriteProcessMemmory()`

and thats it now the iat table is patched and the imports are resolved 

FULL CODE : 
```c
	if (ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress != 0)
	{


		DWORD importRVA = ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress;

		DWORD importSize = ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].Size;

		IMAGE_IMPORT_DESCRIPTOR* importDesc = (PIMAGE_IMPORT_DESCRIPTOR)(BaseAddress + ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);


		SIZE_T descriptorSize = sizeof(IMAGE_IMPORT_DESCRIPTOR);

		PIMAGE_IMPORT_DESCRIPTOR currentDesc = importDesc;
		while (currentDesc->Name)
		{
			const char* dllName = (char*)(BaseAddress + currentDesc->Name);
			HMODULE dllHandle = GetModuleHandleA(dllName);

			if (!dllHandle) dllHandle = LoadLibraryA(dllName);

			PIMAGE_THUNK_DATA origThunk = (PIMAGE_THUNK_DATA)(BaseAddress + currentDesc->OriginalFirstThunk);
			PIMAGE_THUNK_DATA firstThunk = (PIMAGE_THUNK_DATA)(BaseAddress + currentDesc->FirstThunk);
			while (origThunk->u1.AddressOfData)
			{
				FARPROC funcAddress = NULL;
				if (origThunk->u1.Ordinal & IMAGE_ORDINAL_FLAG64)
				{
					WORD ordinal = (WORD)(origThunk->u1.Ordinal & 0xFFFF);
					funcAddress = GetProcAddress(dllHandle, (LPCSTR)(uintptr_t)ordinal);

				}
				else
				{
					PIMAGE_IMPORT_BY_NAME importByName = (PIMAGE_IMPORT_BY_NAME)(BaseAddress + origThunk->u1.AddressOfData);
					char* funcName = (char*)importByName->Name;
					funcAddress = GetProcAddress(dllHandle, funcName);
				
				}
                        ULONGLONG remoteIATEntry = (ULONGLONG)NewAddress + ((BYTE*)firstThunk - (BYTE*)BaseAddress);

				SIZE_T bytesWritten1 = 0;
				if (!WriteProcessMemory(pi.hProcess, (LPVOID)remoteIATEntry, &funcAddress, sizeof(funcAddress), &bytesWritten1)) {
					error("Failed to write to remote IAT");
					return -1;
				}

				origThunk++;
				firstThunk++;
			
			}
			 currentDesc++;


		}
	}
```