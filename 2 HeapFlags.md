* The heap contains *Two Flags (Flags & ForceFlags)* that are initialized in conjunction with the NtGlobalFlag;
* The *Two Flags (Flags & ForceFlags)* locations are different depends on the version of Windows:
    1. Flags:
        * - 0x40 in the heap on 32bit Windows Vista or later;
        * - 0x70 in the heap on 64bit Windows Vista or later;
        * Normally set to HEAP_GROWABLE (2);
        * *When a debugger is present*, on 64bit Windows, Flags is set to:
            ```
            HEAP_GROWABLE (2)
            HEAP_TAIL_CHECKING_ENABLED (0x20)
            HEAP_FREE_CHECKING_ENABLED (0x40)
            HEAP_VALIDATE_PARAMETERS_ENABLED (0x40000000)
            ```
    2. ForceFlags:
        * - 0x44 in the heap on 32bit Windows Vista or later;
        * - 0x74 in the heap on 64bit Windows Vista or later;
        * Normally set to 0;
        * *When a debugger is present*, on 64bit Windows, ForceFlags is set to:
            ```
            HEAP_TAIL_CHECKING_ENABLED (0x20)
            HEAP_FREE_CHECKING_ENABLED (0x40)
            HEAP_VALIDATE_PARAMETERS_ENABLED (0x40000000)
            ```
    * both of these values depend on the *subsystem version* of the host process for a *32-bit process*;
        * Subsystem Version >=3.51:     stated above;
        * Subsystem Version 3.10-3.50:  HEAP_CREATE_ALIGN_16 (0x1000) flag will also be set in both fields;
        * Subsystem Version < 3.10:     the file will not run;
* How it is set?
    * The HEAP_TAIL_CHECKING_ENABLED flag is set in the heap fields if the *FLG_HEAP_ENABLE_TAIL_CHECK* flag is set in the NtGlobalFlag field. 
    * The HEAP_FREE_CHECKING_ENABLED flag is set in the heap fields if the *FLG_HEAP_ENABLE_FREE_CHECK* flag is set in the NtGlobalFlag field. 
    * The HEAP_VALIDATE_PARAMETERS_ENABLED flag (and the HEAP_CREATE_ALIGN_16 (0x10000) flag on Windows NT and Windows 2000) is set in the heap fields if the *FLG_HEAP_VALIDATE_PARAMETERS* flag is set in the NtGlobalFlag field.
    * "PageHeapFlags" string value of the "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\<filename>" registry key.
 * How to retrieve the heap base?
    * 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows:
    ```
    mov eax,    fs:[30h]    ; Process Environment Block
    mov eax,    [eax+18h]   ; get process heap base
    ```
    * 64bit code to examine 64bit Windows environment:
    ```
    push 60h
    pop  rsi
    gs:lodsq                ; Load Qword from address RSI into RAX = Load PEB to RAX
    mov  eax,   [rax+30h]   ; get process heap base
    ```
    * 32bit code to examine 64bit Windows environment:
    ```
    mov  eax,   fs:[30h]    ; PEB
    mov  eax,   [eax+1030h] ; get process heap base
    ```
    * kernel32 GetProcessHeap()
        * forward to ntdll RtlGetProcessHeap() to get process heap list;
        * How to get process heap list?
            1. 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows:
                ```
                push 30h
                pop  esi
                fs:lodsd    ; eax = PEB
                mov  esi,   [esi+eax+5ch]
                lodsd       ; eax = process heaps list base
                ```
            2. 64bit code to examine 64bit Windows environment:
                ```
                push 60h
                pop  rsi
                gs:lodsq    ; rax = PEB
                mov  esi,   [rsi*2 + rax + 20h]
                lodsd       ; eax = process heaps list base
                ```
            3. 32bit code to examine 64bit Windows environment:
                ```
                mov  eax,   fs:[30h]    ; eax = PEB
                mov  esi,   [eax + 10f0h]   
                lodsd       ; eax = process heaps list base
                ```
* How to detect a debugger through *Flags* field?
    * 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows && subsystem version 3.10-3.50:
        ```
            call GetVersion     ; Get major and minor version of OS
                                ; Return: low-order byte = the major version 
            cmp  al,    6       ; al=major version
            cmc                 ; Complement CF flag - CF=!CF     
            sbb  ebx,   ebx     ; subtract with borrow (CF)
                                ; if majorversion<6, CF=1 after cmc, CF=0, ebx = 0
                                ; if majorversion>=6,CF=0 after cmc, CF=1, ebx = -1
            and  ebx,   34h     ; if majorversion>=6, ebx=34h
            mov  eax,   fs:[30h]; eax = PEB
            mov  eax,   [eax+18h]           ; eax = Process Heap Base
            mov  eax,   [eax + ebx + 0ch]   ; eax = Flags
            and  eax,   0effeffffh  ; clear HEAP_CREATE_ALIGN_16 and HEAP_SKIP_VALIDATION_CHECKS
            cmp  eax,   40000062h   ; check HEAP_GROWABLE, HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
            je   being_debugged
        ```
    * 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows && subsystem version > 3.50:
        ```
            call GetVersion     ; Get major and minor version of OS
                                ; Return: low-order byte = the major version 
            cmp  al,    6       ; al=major version
            cmc                 ; Complement CF flag - CF=!CF     
            sbb  ebx,   ebx     ; subtract with borrow (CF)
                                ; if majorversion<6, CF=1 after cmc, CF=0, ebx = 0
                                ; if majorversion>=6,CF=0 after cmc, CF=1, ebx = -1
            and  ebx,   34h     ; if majorversion>=6, ebx=34h
            mov  eax,   fs:[30h]; eax = PEB
            mov  eax,   [eax+18h]           ; eax = Process Heap Base
            mov  eax,   [eax + ebx + 0ch]   ; eax = Flags
            bswap       eax     ;  bswap - reverses the byte order in a 32-bit register operand
            and  al,    0efh    ; clear HEAP_SKIP_VALIDATION_CHECKS
            cmp  eax,   62000040h   ; check HEAP_GROWABLE, HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
            je   being_debugged
        ```
    * 64bit code to examine 64bit Windows environment:
        ```
            push 60h
            pop  rsi
            gs:lodsq    ; rax = PEB
            mov  ebx,   [rax + 30h] ; ebx = Process Heap Base
            call GetVersion
            cmp  al,    6
            sbb  rax,   rax
            and  al,    0a4h    ; if majorversion>=6, al=0a4h
            cmp  d [rbx+rax+70h],   40000062h   ; check HEAP_GROWABLE, HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
            je   being_debugged
        ```
    * 32bit code to examine 64bit Windows environment
        ```
        push 30h
        pop  eax
        mov  ebx,   fs:[eax]    ; ebx = PEB
        mov  ah,    10h
        mov  ebx,   [ebx+eax]   ; ebx = Process Heap Base
        call GetVersion
        cmp  al,    6
        sbb  eax,   eax
        and  al,    0a4h    ; if majorversion>=6, al=0a4h
        cmp  [ebx+eax+70h], 40000062h   ; check HEAP_GROWABLE, HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
        je   being_debugged
        ```
* How to detect a debugger through *ForceFlags* field?
    * 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows && subsystem version 3.10-3.50:
        ```
        call GetVersion     ; Get major and minor version of OS
                            ; Return: low-order byte = the major version 
        cmp  al,    6       ; al=major version
        cmc                 ; Complement CF flag - CF=!CF     
        sbb  ebx,   ebx     ; subtract with borrow (CF)
                            ; if majorversion<6, CF=1 after cmc, CF=0, ebx = 0
                            ; if majorversion>=6,CF=0 after cmc, CF=1, ebx = -1
        and  ebx,   34h     ; if majorversion>=6, ebx=34h
        mov  eax,   fs:[30h]; eax = PEB
        mov  eax,   [eax+18h]           ; eax = Process Heap Base
        mov  eax,   [eax + ebx + 10h]   ; eax = ForceFlags
        btr  eax,   10h     ; btr - CF = Selected Bit, Selected Bit = 0
                            ; CF = eax bit 10h, eax bit 10h = 0
        cmp  eax,   40000060h   ; check HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
        je   being_debugged
        ```
    * 32bit code to examine 32bit Windows environment on 32bit or 64bit versions of Windows && subsystem version > 3.50:
        ```
        call GetVersion     ; Get major and minor version of OS
                            ; Return: low-order byte = the major version 
        cmp  al,    6       ; al=major version
        cmc                 ; Complement CF flag - CF=!CF     
        sbb  ebx,   ebx     ; subtract with borrow (CF)
                            ; if majorversion<6, CF=1 after cmc, CF=0, ebx = 0
                            ; if majorversion>=6,CF=0 after cmc, CF=1, ebx = -1
        and  ebx,   34h     ; if majorversion>=6, ebx=34h
        mov  eax,   fs:[30h]; eax = PEB
        mov  eax,   [eax+18h]           ; eax = Process Heap Base
        cmp  [eax+ebx+10h],   40000060h   ; check HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
        je   being_debugged
        ```
    * 64bit code to examine 64bit Windows environment:
        ```
        push 60h
        pop  rsi
        gs:lodsq    ; rax = PEB
        mov  ebx,   [rax + 30h] ; ebx = Process Heap Base
        call GetVersion
        cmp  al,    6
        sbb  rax,   rax
        and  al,    0a4h    ; if majorversion>=6, al=0a4h
        cmp  d [rbx+rax+74h],   40000060h   ; check HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
        je   being_debugged
        ```
    * 32bit code to examine 64bit Windows environment
        ```
        call GetVersion
        cmp  al,    6
        push 30h
        pop  eax
        mov  ebx,   fs:[eax]    ; ebx = PEB
        mov  ah,    10h
        mov  ebx,   [ebx+eax]   ; ebx = Process Heap base
        sbb  eax,   eax
        and  al,    0a4h    ; if majorversion>=6, al=0a4h
        cmp  [ebx+eax+74h], 40000060h   ; check HEAP_TAIL_CHECKING_ENABLED, HEAP_FREE_CHECKING_ENABLED, HEAP_VALIDATE_PARAMETERS_ENABLED
        je   being_debugged
        ```
        








