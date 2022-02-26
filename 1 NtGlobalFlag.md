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
    * 64bits windows:
    ```asm
    push 60h
    pop rsi
    gs:lodsq    ; Load Qword from address rsi into RAX
    mov al, [rsi*2+rax-14h] ; NtGlobalFlag
    and al, 70h
    cmp al, 70h
    je  being_debugged
    ```
    * 32bit code on 64bits windows
    ```asm
    mov eax, fs:[30h]   ; PEB
    mov al, [eax + 10bch]   ; NtGlobalFlag
    and al, 70h
    cmp al, 70h
    je  being_debugged
* Defeat:
    1. "GlobalFlag" string value of the "HKLM\System\CurrentControlSet\Control\SessionManager" registry key.
        * How it work? - at the system start, the NtGlobalFlag value is initilized with the value from the registry value;
        * Affects all processes in system;
        * Requires reboot to take effect;
        * May be changed by Windows later;
    2. "GlobalFlag" string value "HKLM\Software\Microsoft\WindowsNT\CurrentVersion\Image File Execution Options\<filename>" registry key
        * How it work? - at the system start, the NtGlobalFlag value is initilized with the value from the registry value;
    3. GlobalFlagsClear and GlobalFlagsSet in Load Configuration Table;
        * GlobalFlagsClear  - list the flags to clear;
        * GlobalFlagsSet    - list the flags to set;
        * The 2 fields are applied after GlobalFlag has been applied to NtGlobalFlags;
    4. Setting the "_NO_DEBUG_HEAP" environment variable, the three heap flags will not be set in the NtGlobalFlag field because of the debugger.
