* Theory:
    * If HEAP_TAIL_CHECKING_ENABLED flag is set, then **0xABABABAB** will be appended twice in a 32-bit Windows environment / four times in a 64-bit Windows;
    * If the HEAP_FREE_CHECKING_ENABLED flag is set, then **0xFEEEFEEE** (or a part thereof) will be appended if additional bytes are required to fill in;
the slack space until the next block.
environment) at the exact end of the allocated block.
* How to detect?
    * 32-bit code to examine the 32-bit Windows environment on 32-bit or 64-bit versions of Windows:
        ```
        xor     ebx,    ebx
        call    GetVersion
        cmp     al,     6
        sbb     ebp,    ebp     ; if al > 6, ebp = 0ffff ffff h
        jb      l1
        mov     eax,    fs:[ebx+30h]    ; eax = PEB
        mov     eax,    [eax + 18h]     ; eax = Process Heap Base
        mov     ecx,    [eax + 24h]     ; check for Heap Protection
        jecxz   l1                      ; if ecx == 0, jmp l1
        mov     ecx,    [ecx]
        cmovne  ebx,    [eax+50h]       ; cmovne - mov if ZF = 0 ???
                                        ; conditionally get heap key
    l1: mov     eax,    heap ptr
        movzx   edx,    w [eax - 8]     ; size, movzx - mov and zero extend
        xor     dx,     bx
        movzx   ecx,    b [eax + ebp - 1]
        sub     eax,    ecx
        lea     edi,    [edx*8 + eax]
        mov     al,     0abh
        mov     cl,     8
        repe    scasb   ; repe  - repeat while equal
                        ; scasb - scan byte string, compare AL with ES:DI/EDI/RDI then set status flags