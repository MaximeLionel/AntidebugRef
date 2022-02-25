* NtGlobalFlag is in PEB structure
* Location (PEB): 
    * 0x68 of 32-bit Windows
    * 0xBC of 64-bit Windows
* Value: 
    * 0 - Not Being Debugged.
    * 0x70 - Being Debugged.
        * FLG_HEAP_ENABLE_TAIL_CHECK  - 0x10 
        * FLG_HEAP_ENABLE_FREE_CHECK  - 0x20 
        * FLG_HEAP_VALIDATE_PARAMETERS-	0x40 
        * Total	- 0x70 
* Detect a debugger:
    * 32bits windows:
    ```asm
    mov eax, fs:[30h] ;Process Environment Block
     mov al, [eax+68h] ;NtGlobalFlag
    and al, 70h
    cmp al, 70h
    je being_debugged
    ```