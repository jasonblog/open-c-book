# 緩衝區溢出與注入分析

-    [前言](#toc_14869_27504_1)
-    [進程的內存映像](#toc_14869_27504_2)
    -    [常用寄存器初識](#toc_14869_27504_3)
    -    [call，ret 指令的作用分析](#toc_14869_27504_4)
    -    [什麼是系統調用](#toc_14869_27504_5)
    -    [什麼是 ELF 文件](#toc_14869_27504_6)
    -    [程序執行基本過程](#toc_14869_27504_7)
    -    [Linux 下程序的內存映像](#toc_14869_27504_8)
    -    [棧在內存中的組織](#toc_14869_27504_9)
-    [緩衝區溢出](#toc_14869_27504_10)
    -    [實例分析：字符串複製](#toc_14869_27504_11)
    -    [緩衝區溢出後果](#toc_14869_27504_12)
    -    [緩衝區溢出應對策略](#toc_14869_27504_13)
    -    [如何保護 ebp 不被修改](#toc_14869_27504_14)
    -    [如何保護 eip 不被修改？](#toc_14869_27504_15)
    -    [緩衝區溢出檢測](#toc_14869_27504_16)
-    [緩衝區注入實例](#toc_14869_27504_17)
    -    [準備：把 C 語言函數轉換為字符串序列](#toc_14869_27504_18)
    -    [注入：在 C 語言中執行字符串化的代碼](#toc_14869_27504_19)
    -    [注入原理分析](#toc_14869_27504_20)
    -    [緩衝區注入與防範](#toc_14869_27504_21)
-    [後記](#toc_14869_27504_22)
-    [參考資料](#toc_14869_27504_23)


<span id="toc_14869_27504_1"></span>
## 前言

雖然程序加載以及動態符號鏈接都已經很理解了，但是這夥卻被進程的內存映像給”糾纏"住。看著看著就一發不可收拾——很有趣。

下面一起來探究“緩衝區溢出和注入”問題（主要是關心程序的內存映像）。

<span id="toc_14869_27504_2"></span>
## 進程的內存映像

永遠的 `Hello World`，太熟悉了吧，

```
#include <stdio.h>
int main(void)
{
	printf("Hello World\n");
	return 0;
}
```

如果要用內聯彙編（`inline assembly`）來寫呢？

```
 1  /* shellcode.c */
 2  void main()
 3  {
 4      __asm__ __volatile__("jmp forward;"
 5                   "backward:"
 6                           "popl   %esi;"
 7                           "movl   $4, %eax;"
 8                           "movl   $2, %ebx;"
 9                           "movl   %esi, %ecx;"
10                           "movl   $12, %edx;"
11                           "int    $0x80;"	/* system call 1 */
12                           "movl   $1, %eax;"
13                           "movl   $0, %ebx;"
14                           "int    $0x80;"	/* system call 2 */
15                   "forward:"
16                           "call   backward;"
17                           ".string \"Hello World\\n\";");
18  }
```

看起來很複雜，實際上就做了一個事情，往終端上寫了個 `Hello World` 。不過這個非常有意思。先簡單分析一下流程：

- 第 4 行指令的作用是跳轉到第 15 行（即 `forward` 標記處），接著執行第 16 行。
- 第 16 行調用 `backward`，跳轉到第 5 行，接著執行 6 到 14 行。
- 第 6 行到第 11 行負責在終端打印出 `Hello World` 字符串（等一下詳細介紹）。
- 第 12 行到第 14 行退出程序（等一下詳細介紹）。

為了更好的理解上面的代碼和後續的分析，先來介紹幾個比較重要的內容。

<span id="toc_14869_27504_3"></span>
### 常用寄存器初識

`X86` 處理器平臺有三個常用寄存器：程序指令指針、程序堆棧指針與程序基指針：

|寄存器|名稱        |註釋                      |
|------|------------|--------------------------|
|EIP   |程序指令指針|通常指向下一條指令的位置  |
|ESP   |程序堆棧指針|通常指向當前堆棧的當前位置|
|EBP   |程序基指針  |通常指向函數使用的堆棧頂端|

當然，上面都是擴展的寄存器，用於 32 位系統，對應的 16 系統為 `ip`，`sp`，`bp` 。

<span id="toc_14869_27504_4"></span>
### call，ret 指令的作用分析

- call` 指令

    跳轉到某個位置，並在之前把下一條指令的地址（`EIP`）入棧（為了方便”程序“返回以後能夠接著執行）。這樣的話就有：

        call backward   ==>   push eip
                              jmp backward

- ret 指令

    通常 `call` 指令和 `ret` 是配合使用的，前者壓入跳轉前的下一條指令地址，後者彈出 `call` 指令壓入的那條指令，從而可以在函數調用結束以後接著執行後面的指令。

        ret                    ==>   pop eip

通常在函數調用後，還需要恢復 `esp` 和 `ebp`，恢復 `esp` 即恢復當前棧指針，以便釋放調用函數時為存儲函數的局部變量而自動分配的空間；恢復 `ebp` 是從棧中彈出一個數據項（通常函數調用過後的第一條語句就是 `push ebp`），從而恢復當前的函數指針為函數調用者本身。這兩個動作可以通過一條 `leave` 指令完成。

這三個指令對我們後續的解釋會很有幫助。更多關於 Intel 的指令集，請參考：[Intel 386 Manual](http://www.x86.org/intel.doc/386manuals.htm), x86 Assembly Language FAQ：[part1](http://www.faqs.org/faqs/assembly-language/x86/general/part1/), [part2](http://www.faqs.org/faqs/assembly-language/x86/general/part2/), [part3](http://www.faqs.org/faqs/assembly-language/x86/general/part3/).

<span id="toc_14869_27504_5"></span>
### 什麼是系統調用（以 Linux 2.6.21 版本和 x86 平臺為例）

系統調用是用戶和內核之間的接口，用戶如果想寫程序，很多時候直接調用了 C 庫，並沒有關心繫統調用，而實際上 C 庫也是基於系統調用的。這樣應用程序和內核之間就可以通過系統調用聯繫起來。它們分別處於操作系統的用戶空間和內核空間（主要是內存地址空間的隔離）。

```
用戶空間         應用程序(Applications)
                        |      |
                        |     C庫（如glibc）
                        |      |
                       系統調用(System Calls，如sys_read, sys_write, sys_exit)
                            |
內核空間              內核(Kernel)
```

系統調用實際上也是一些函數，它們被定義在 `arch/i386/kernel/sys_i386.c` （老的在 `arch/i386/kernel/sys.c`）文件中，並且通過一張系統調用表組織，該表在內核啟動時就已經加載了，這個表的入口在內核源代碼的 `arch/i386/kernel/syscall_table.S` 裡頭（老的在 `arch/i386/kernel/entry.S`）。這樣，如果想添加一個新的系統調用，修改上面兩個內核中的文件，並重新編譯內核就可以。當然，如果要在應用程序中使用它們，還得把它寫到 `include/asm/unistd.h` 中。

如果要在 C 語言中使用某個系統調用，需要包含頭文件 `/usr/include/asm/unistd.h`，裡頭有各個系統調用的聲明以及系統調用號（對應於調用表的入口，即在調用表中的索引，為方便查找調用表而設立的）。如果是自己定義的新系統調用，可能還要在開頭用宏 `_syscall(type, name, type1, name1...)`來聲明好參數。

如果要在彙編語言中使用，需要用到 `int 0x80` 調用，這個是系統調用的中斷入口。涉及到傳送參數的寄存器有這麼幾個，`eax` 是系統調用號（可以到 `/usr/include/asm-i386/unistd.h` 或者直接到 `arch/i386/kernel/syscall_table.S` 查到），其他寄存器如 `ebx`，`ecx`，`edx`，`esi`，`edi` 一次存放系統調用的參數。而系統調用的返回值存放在 `eax` 寄存器中。

下面我們就很容易解釋前面的 `Shellcode.c` 程序流程的 2，3 兩部分了。因為都用了 `int 0x80` 中斷，所以都用到了系統調用。

第 3 部分很簡單，用到的系統調用號是 1，通過查表（查 `/usr/include/asm-i386/unistd.h` 或 `arch/i386/kernel/syscall_table.S`）可以發現這裡是 `sys_exit` 調用，再從 `/usr/include/unistd.h` 文件看這個系統調用的聲明，發現參數 `ebx` 是程序退出狀態。

第 2 部分比較有趣，而且複雜一點。我們依次來看各個寄存器，首先根據 `eax` 為 4 確定（同樣查表）系統調用為 `sys_write`，而查看它的聲明（從 `/usr/include/unistd.h`），我們找到了參數依次為文件描述符、字符串指針和字符串長度。

- 第一個參數是 `ebx`，正好是 2，即標準錯誤輸出，默認為終端。
- 第二個參數是 `ecx`，而 `ecx` 的內容來自 `esi`，`esi` 來自剛彈出棧的值（見第 6 行 `popl %esi;`），而之前剛好有 `call` 指令引起了最近一次壓棧操作，入棧的內容剛好是 `call` 指令的下一條指令的地址，即 `.string` 所在行的地址，這樣 `ecx` 剛好引用了 `Hello World\\n` 字符串的地址。
- 第三個參數是 `edx`，剛好是 12，即 `Hello World\\n` 字符串的長度（包括一個空字符）。這樣，`Shellcode.c` 的執行流程就很清楚了，第 4，5，15，16 行指令的巧妙之處也就容易理解了（把 `.string` 存放在 `call` 指令之後，並用 `popl` 指令把 `eip` 彈出當作字符串的入口）。

<span id="toc_14869_27504_6"></span>
### 什麼是 ELF 文件

這裡的 ELF 不是“精靈”，而是 Executable and Linking Format 文件，是 Linux 下用來做目標文件、可執行文件和共享庫的一種文件格式，它有專門的標準，例如：[X86 ELF format and ABI](http://www.x86.org/ftp/manuals/tools/elf.pdf)，[中文版](http://www.xfocus.net/articles/200105/174.html)。

下面簡單描述 `ELF` 的格式。

`ELF` 文件主要有三種，分別是：

- 可重定位的目標文件，在編譯時用 `gcc` 的 `-c` 參數時產生。
- 可執行文件，這類文件就是我們後面要討論的可以執行的文件。
- 共享庫，這裡主要是動態共享庫，而靜態共享庫則是可重定位的目標文件通過 `ar` 命令組織的。

`ELF` 文件的大體結構：

```
ELF Header               #程序頭，有該文件的Magic number(參考man magic)，類型等
Program Header Table     #對可執行文件和共享庫有效，它描述下面各個節(section)組成的段
Section1
Section2
Section3
.....
Program Section Table   #僅對可重定位目標文件和靜態庫有效，用於描述各個Section的重定位信息等。
```

對於可執行文件，文件最後的 `Program Section Table` （節區表）和一些非重定位的 `Section`，比如 `.comment`，`.note.XXX.debug` 等信息都可以刪除掉，不過如果用 `strip`，`objcopy` 等工具刪除掉以後，就不可恢復了。因為這些信息對程序的運行一般沒有任何用處。

`ELF` 文件的主要節區（`section`）有 `.data`，`.text`，`.bss`，`.interp` 等，而主要段（`segment`）有 `LOAD`，`INTERP` 等。它們之間（節區和段）的主要對應關係如下：

|Section |解釋                                      | 實例                |
|--------|------------------------------------------|---------------------|
|.data   |初始化的數據                              | 比如 `int a=10`     |
|.bss    |未初始化的數據                            | 比如 `char sum[100];` 這個在程序執行之前，內核將初始化為 0 |
|.text   |程序代碼正文                              | 即可執行指令集      |
|.interp |描述程序需要的解釋器（動態連接和裝載程序）| 存有解釋器的全路徑，如 `/lib/ld-linux.so` |

而程序在執行以後，`.data`，`.bss`，`.text` 等一些節區會被 `Program header table` 映射到 `LOAD` 段，`.interp` 則被映射到了 `INTERP` 段。

對於 `ELF` 文件的分析，建議使用 `file`，`size`，`readelf`，`objdump`，`strip`，`objcopy`，`gdb`，`nm` 等工具。

這裡簡單地演示這幾個工具：

```
$ gcc -g -o shellcode shellcode.c  #如果要用gdb調試，編譯時加上-g是必須的
shellcode.c: In function ‘main’:
shellcode.c:3: warning: return type of ‘main’ is not ‘int’
f$ file shellcode  #file命令查看文件類型，想了解工作原理，可man magic,man file
shellcode: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV),
dynamically linked (uses shared libs), not stripped
$ readelf -l shellcode  #列出ELF文件前面的program head table，後面是它描
                       #述了各個段(segment)和節區(section)的關係,即各個段包含哪些節區。
Elf file type is EXEC (Executable file)
Entry point 0x8048280
There are 7 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x000e0 0x000e0 R E 0x4
  INTERP         0x000114 0x08048114 0x08048114 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x0044c 0x0044c R E 0x1000
  LOAD           0x00044c 0x0804944c 0x0804944c 0x00100 0x00104 RW  0x1000
  DYNAMIC        0x000460 0x08049460 0x08049460 0x000c8 0x000c8 RW  0x4
  NOTE           0x000128 0x08048128 0x08048128 0x00020 0x00020 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .hash .dynsym .dynstr .gnu.version .gnu.version_r
          .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame
   03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag
   06
$ size shellcode   #可用size命令查看各個段（對應後面將分析的進程內存映像）的大小
   text    data     bss     dec     hex filename
    815     256       4    1075     433 shellcode
$ strip -R .note.ABI-tag shellcode #可用strip來給可執行文件“減肥”，刪除無用信息
$ size shellcode               #“減肥”後效果“明顯”，對於嵌入式系統應該有很大的作用
   text    data     bss     dec     hex filename
    783     256       4    1043     413 shellcode
$ objdump -s -j .interp shellcode #這個主要工作是反編譯，不過用來查看各個節區也很厲害

shellcode:     file format elf32-i386

Contents of section .interp:
 8048114 2f6c6962 2f6c642d 6c696e75 782e736f  /lib/ld-linux.so
 8048124 2e3200                               .2.
```

補充：如果要刪除可執行文件的 `Program Section Table`，可以用 [A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux](http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html) 一文的作者寫的 [elf kicker](http://www.muppetlabs.com/~breadbox/software/elfkickers.html) 工具鏈中的 `sstrip` 工具。

<span id="toc_14869_27504_7"></span>
### 程序執行基本過程

在命令行下，敲入程序的名字或者是全路徑，然後按下回車就可以啟動程序，這個具體是怎麼工作的呢？

首先要再認識一下我們的命令行，命令行是內核和用戶之間的接口，它本身也是一個程序。在 Linux 系統啟動以後會為每個終端用戶建立一個進程執行一個 Shell 解釋程序，這個程序解釋並執行用戶輸入的命令，以實現用戶和內核之間的接口。這類解釋程序有哪些呢？目前 Linux 下比較常用的有 `/bin/bash` 。那麼該程序接收並執行命令的過程是怎麼樣的呢？

先簡單描述一下這個過程：

- 讀取用戶由鍵盤輸入的命令行。
- 分析命令，以命令名作為文件名，並將其它參數改為系統調用 `execve` 內部處理所要求的形式。
- 終端進程調用 `fork` 建立一個子進程。
- 終端進程本身用系統調用 `wait4` 來等待子進程完成（如果是後臺命令，則不等待）。當子進程運行時調用 `execve`，子進程根據文件名（即命令名）到目錄中查找有關文件（這是命令解釋程序構成的文件），將它調入內存，執行這個程序（解釋這條命令）。
- 如果命令末尾有 `&` 號（後臺命令符號），則終端進程不用系統調用 `wait4` 等待，立即發提示符，讓用戶輸入下一個命令，轉 1）。如果命令末尾沒有 `&` 號，則終端進程要一直等待，當子進程（即運行命令的進程）完成處理後終止，向父進程（終端進程）報告，此時終端進程醒來，在做必要的判別等工作後，終端進程發提示符，讓用戶輸入新的命令，重複上述處理過程。

現在用 `strace` 來跟蹤一下程序執行過程中用到的系統調用。

```
$ strace -f -o strace.out test
$ cat strace.out | grep \(.*\) | sed -e "s#[0-9]* \([a-zA-Z0-9_]*\)(.*).*#\1#g"
execve
brk
access
open
fstat64
mmap2
close
open
read
fstat64
mmap2
mmap2
mmap2
mmap2
close
mmap2
set_thread_area
mprotect
munmap
brk
brk
open
fstat64
mmap2
close
close
close
exit_group
```

相關的系統調用基本體現了上面的執行過程，需要注意的是，裡頭還涉及到內存映射（`mmap2`）等。

下面再羅嗦一些比較有意思的內容，參考《深入理解 Linux 內核》的程序的執行（P681）。

Linux 支持很多不同的可執行文件格式，這些不同的格式是如何解釋的呢？平時我們在命令行下敲入一個命令就完了，也沒有去管這些細節。實際上 Linux 下有一個 `struct linux_binfmt` 結構來管理不同的可執行文件類型，這個結構中有對應的可執行文件的處理函數。大概的過程如下：

- 在用戶態執行了 `execve` 後，引發 `int 0x80` 中斷，進入內核態，執行內核態的相應函數 `do_sys_execve`，該函數又調用 `do_execve` 函數。 `do_execve` 函數讀入可執行文件，檢查權限，如果沒問題，繼續讀入可執行文件需要的相關信息（`struct linux_binprm` 描述的）。

- 接著執行 `search_binary_handler`，根據可執行文件的類型（由上一步的最後確定），在 `linux_binfmt` 結構鏈表（`formats`，這個鏈表可以通過 `register_binfmt` 和 `unregister_binfmt` 註冊和刪除某些可執行文件的信息，因此註冊新的可執行文件成為可能，後面再介紹）上查找，找到相應的結構，然後執行相應的 `load_binary` 函數開始加載可執行文件。在該鏈表的最後一個元素總是對解釋腳本（`interpreted script`）的可執行文件格式進行描述的一個對象。這種格式只定義了 `load_binary` 方法，其相應的 `load_script` 函數檢查這種可執行文件是否以兩個 `#!` 字符開始，如果是，這個函數就以另一個可執行文件的路徑名作為參數解釋第一行的其餘部分，並把腳本文件名作為參數傳遞以執行這個腳本（實際上腳本程序把自身的內容當作一個參數傳遞給瞭解釋程序（如 `/bin/bash`），而這個解釋程序通常在腳本文件的開頭用 `#!` 標記，如果沒有標記，那麼默認解釋程序為當前 `SHELL`）。

- 對於 `ELF` 類型文件，其處理函數是 `load_elf_binary`，它先讀入 `ELF` 文件的頭部，根據頭部信息讀入各種數據，再次掃描程序段描述表（`Program Header Table`），找到類型為 `PT_LOAD` 的段（即 `.text`，`.data`，`.bss` 等節區），將其映射（`elf_map`）到內存的固定地址上，如果沒有動態連接器的描述段，把返回的入口地址設置成應用程序入口。完成這個功能的是 `start_thread`，它不啟動一個線程，而只是用來修改了 `pt_regs` 中保存的 `PC` 等寄存器的值，使其指向加載的應用程序的入口。當內核操作結束，返回用戶態時接著就執行應用程序本身了。

- 如果應用程序使用了動態連接庫，內核除了加載指定的可執行文件外，還要把控制權交給動態連接器（`ld-linux.so`）以便處理動態連接的程序。內核搜尋段表（`Program Header Table`），找到標記為 `PT_INTERP` 段中所對應的動態連接器的名稱，並使用 `load_elf_interp` 加載其映像，並把返回的入口地址設置成 `load_elf_interp` 的返回值，即動態鏈接器的入口。當 `execve` 系統調用退出時，動態連接器接著運行，它檢查應用程序對共享鏈接庫的依賴性，並在需要時對其加載，對程序的外部引用進行重定位（具體過程見[《進程和進程的基本操作》][100]）。然後把控制權交給應用程序，從 `ELF` 文件頭部中定義的程序進入點（用 `readelf -h` 可以出看到，`Entry point address` 即是）開始執行。（不過對於非 `LIB_BIND_NOW` 的共享庫裝載是在有外部引用請求時才執行的）。

[100]: 02-chapter7.markdown

對於內核態的函數調用過程，沒有辦法通過 `strace`（它只能跟蹤到系統調用層）來做的，因此要想跟蹤內核中各個系統調用的執行細節，需要用其他工具。比如可以通過 Ftrace 來跟蹤內核具體調用了哪些函數。當然，也可以通過 `ctags/cscope/LXR` 等工具分析內核的源代碼。

Linux 允許自己註冊我們自己定義的可執行格式，主要接口是 `/procy/sys/fs/binfmt_misc/register`，可以往裡頭寫入特定格式的字符串來實現。該字符串格式如下：
`:name:type:offset:string:mask:interpreter:`

- `name` 新格式的標示符
- `type` 識別類型（`M` 表示魔數，`E` 表示擴展）
- `offset` 魔數（`magic number`，請參考 `man magic` 和 `man file`）在文件中的啟始偏移量
- `string` 以魔數或者以擴展名匹配的字節序列
- `mask` 用來屏蔽掉 `string` 的一些位
- `interpreter` 程序解釋器的完整路徑名

<span id="toc_14869_27504_8"></span>
### Linux 下程序的內存映像

Linux 下是如何給進程分配內存（這裡僅討論虛擬內存的分配）的呢？可以從 `/proc/<pid>/maps` 文件中看到個大概。這裡的 `pid` 是進程號。

`/proc` 下有一個文件比較特殊，是 `self`，它鏈接到當前進程的進程號，例如：

```
$ ls /proc/self -l
lrwxrwxrwx 1 root root 64 2000-01-10 18:26 /proc/self -> 11291/
$ ls /proc/self -l
lrwxrwxrwx 1 root root 64 2000-01-10 18:26 /proc/self -> 11292/
```

看到沒？每次都不一樣，這樣我們通過 `cat /proc/self/maps` 就可以看到 `cat` 程序執行時的內存映像了。

```
$ cat -n /proc/self/maps
     1  08048000-0804c000 r-xp 00000000 03:01 273716     /bin/cat
     2  0804c000-0804d000 rw-p 00003000 03:01 273716     /bin/cat
     3  0804d000-0806e000 rw-p 0804d000 00:00 0          [heap]
     4  b7b90000-b7d90000 r--p 00000000 03:01 87528      /usr/lib/locale/locale-archive
     5  b7d90000-b7d91000 rw-p b7d90000 00:00 0
     6  b7d91000-b7ecd000 r-xp 00000000 03:01 466875     /lib/libc-2.5.so
     7  b7ecd000-b7ece000 r--p 0013c000 03:01 466875     /lib/libc-2.5.so
     8  b7ece000-b7ed0000 rw-p 0013d000 03:01 466875     /lib/libc-2.5.so
     9  b7ed0000-b7ed4000 rw-p b7ed0000 00:00 0
    10  b7eeb000-b7f06000 r-xp 00000000 03:01 402817     /lib/ld-2.5.so
    11  b7f06000-b7f08000 rw-p 0001b000 03:01 402817     /lib/ld-2.5.so
    12  bfbe3000-bfbf8000 rw-p bfbe3000 00:00 0          [stack]
    13  ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
```

編號是原文件裡頭沒有的，為了說明方便，用 `-n` 參數加上去的。我們從中可以得到如下信息：

- 第 1，2 行對應的內存區是我們的程序（包括指令，數據等）
- 第 3 到 12 行對應的內存區是堆棧段，裡頭也映像了程序引用的動態連接庫
- 第 13 行是內核空間

總結一下：

- 前兩部分是用戶空間，可以從 `0x00000000` 到 `0xbfffffff` （在測試的 `2.6.21.5-smp` 上只到 `bfbf8000`），而內核空間從 `0xC0000000` 到 `0xffffffff`，分別是 `3G` 和 `1G`，所以對於每一個進程來說，共佔用 `4G` 的虛擬內存空間
- 從程序本身佔用的內存，到堆棧段（動態獲取內存或者是函數運行過程中用來存儲局部變量、參數的空間，前者是 `heap`，後者是 `stack`），再到內核空間，地址是從低到高的
- 棧頂並非 `0xC0000000` 下的一個固定數值

結合相關資料，可以得到這麼一個比較詳細的進程內存映像表（以 `Linux 2.6.21.5-smp` 為例）：

|地址       |   內核空間                       | 描述                                             |
|-----------|----------------------------------|--------------------------------------------------|
|0xC0000000 |                                  |                                                  |
|           |  (program flie) 程序名           | execve 的第一個參數                              |
|           |  (environment) 環境變量          | execve 的第三個參數，main 的第三個參數           |
|           |  (arguments) 參數                | execve 的第二個參數，main 的形參                 |
|           |  (stack) 棧                      | 自動變量以及每次函數調用時所需保存的信息都       |
|           |                                  | 存放在此，包括函數返回地址、調用者的             |
|           |                                  | 環境信息等，函數的參數，局部變量都存放在此       |
|           |  (shared memory) 共享內存        | 共享內存的大概位置                               |
|           |      ...                         |                                                  |
|           |      ...                         |                                                  |
|           |  (heap) 堆                       | 主要在這裡進行動態存儲分配，比如 malloc，new 等。|
|           |      ...                         |                                                  |
|           |  .bss (uninitilized data)        | 沒有初始化的數據（全局變量哦）                   |
|           |  .data (initilized global data)  | 已經初始化的全局數據（全局變量）                 |
|           |  .text (Executable Instructions) | 通常是可執行指令                                 |
|0x08048000 |                                  |                                                  |
|0x00000000 |                                  | ...                                              |


光看沒有任何概念，我們用 `gdb` 來看看剛才那個簡單的程序。

```
$ gcc -g -o shellcode shellcode.c   #要用gdb調試，在編譯時需要加-g參數
$ gdb ./shellcode
(gdb) set args arg1 arg2 arg3 arg4  #為了測試，設置幾個參數
(gdb) l                             #瀏覽代碼
1 /* shellcode.c */
2 void main()
3 {
4     __asm__ __volatile__("jmp forward;"
5     "backward:"
6        "popl   %esi;"
7        "movl   $4, %eax;"
8        "movl   $2, %ebx;"
9        "movl   %esi, %ecx;"
10               "movl   $12, %edx;"
(gdb) break 4               #在彙編入口設置一個斷點，讓程序運行後停到這裡
Breakpoint 1 at 0x8048332: file shellcode.c, line 4.
(gdb) r                     #運行程序
Starting program: /mnt/hda8/Temp/c/program/shellcode arg1 arg2 arg3 arg4

Breakpoint 1, main () at shellcode.c:4
4     __asm__ __volatile__("jmp forward;"
(gdb) print $esp            #打印當前堆棧指針值，用於查找整個棧的棧頂
$1 = (void *) 0xbffe1584
(gdb) x/100s $esp+4000      #改變後面的4000，不斷往更大的空間找
(gdb) x/1s 0xbffe1fd9       #在 0xbffe1fd9 找到了程序名，這裡是該次運行時的棧頂
0xbffe1fd9:      "/mnt/hda8/Temp/c/program/shellcode"
(gdb) x/10s 0xbffe17b7      #其他環境變量信息
0xbffe17b7:      "CPLUS_INCLUDE_PATH=/usr/lib/qt/include"
0xbffe17de:      "MANPATH=/usr/local/man:/usr/man:/usr/X11R6/man:/usr/lib/java/man:/usr/share/texmf/man"
0xbffe1834:      "HOSTNAME=falcon.lzu.edu.cn"
0xbffe184f:      "TERM=xterm"
0xbffe185a:      "SSH_CLIENT=219.246.50.235 3099 22"
0xbffe187c:      "QTDIR=/usr/lib/qt"
0xbffe188e:      "SSH_TTY=/dev/pts/0"
0xbffe18a1:      "USER=falcon"
...
(gdb) x/5s 0xbffe1780    #一些傳遞給main函數的參數，包括文件名和其他參數
0xbffe1780:      "/mnt/hda8/Temp/c/program/shellcode"
0xbffe17a3:      "arg1"
0xbffe17a8:      "arg2"
0xbffe17ad:      "arg3"
0xbffe17b2:      "arg4"
(gdb) print init  #打印init函數的地址，這個是/usr/lib/crti.o裡頭的函數，做一些初始化操作
$2 = {<text variable, no debug info>} 0xb7e73d00 <init>
(gdb) print fini   #也在/usr/lib/crti.o中定義，在程序結束時做一些處理工作
$3 = {<text variable, no debug info>} 0xb7f4a380 <fini>
(gdb) print _start #在/usr/lib/crt1.o，這個才是程序的入口，必須的，ld會檢查這個
$4 = {<text variable, no debug info>} 0x8048280 <__libc_start_main@plt+20>
(gdb) print main   #這裡是我們的main函數
$5 = {void ()} 0x8048324 <main>
```

補充：在進程的內存映像中可能看到諸如 `init`，`fini`，`_start` 等函數（或者是入口），這些東西並不是我們自己寫的啊？為什麼會跑到我們的代碼裡頭呢？實際上這些東西是鏈接的時候 `gcc` 默認給連接進去的，主要用來做一些進程的初始化和終止的動作。更多相關的細節可以參考資料[如何獲取當前進程之靜態影像文件](http://edu.stuccess.com/KnowCenter/Unix/13/hellguard_unix_faq/00000089.htm)和"The Linux Kernel Primer"， P234， Figure 4.11，如果想了解鏈接（ld）的具體過程，可以看看本節參考《Unix環境高級編程編程》第7章 "UnIx進程的環境"， P127和P13，[ELF: From The Programmer's Perspective](http://linux.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html)，[GNU-ld 連接腳本 Linker Scripts](http://womking.bokee.com/5967668.html)。

上面的操作對堆棧的操作比較少，下面我們用一個例子來演示棧在內存中的情況。

<span id="toc_14869_27504_9"></span>
### 棧在內存中的組織

這一節主要介紹一個函數被調用時，參數是如何傳遞的，局部變量是如何存儲的，它們對應的棧的位置和變化情況，從而加深對棧的理解。在操作時發現和參考資料的結果不太一樣（參考資料中沒有 `edi` 和 `esi` 相關信息，再第二部分的一個小程序裡頭也沒有），可能是 `gcc` 版本的問題或者是它對不同源代碼的處理不同。我的版本是 `4.1.2` （可以通過 `gcc --version` 查看）。

先來一段簡單的程序，這個程序除了做一個加法操作外，還複製了一些字符串。

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h>     /* memset, memcpy */

#define BUF_SIZE 8

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

int func(int a, int b, int c)
{
	int sum = 0;
	char buffer[BUF_SIZE];

	sum = a + b + c;

	memset(buffer, '\0', BUF_SIZE);
	memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

	return sum;
}

int main()
{
	int sum;

	sum = func(1, 2, 3);

	printf("sum = %d\n", sum);

	return 0;
}
```

上面這個代碼沒有什麼問題，編譯執行一下：

```
$ make testshellcode
cc     testshellcode.c   -o testshellcode
$ ./testshellcode
sum = 6
```

下面調試一下，看看在調用 `func` 後的棧的內容。

```
$ gcc -g -o testshellcode testshellcode.c  #為了調試，需要在編譯時加-g選項
$ gdb ./testshellcode   #啟動gdb調試
...
(gdb) set logging on    #如果要記錄調試過程中的信息，可以把日誌記錄功能打開
Copying output to gdb.txt.
(gdb) l main            #列出源代碼
20
21              return sum;
22      }
23
24      int main()
25      {
26              int sum;
27
28              sum = func(1, 2, 3);
29
(gdb) break 28   #在調用func函數之前讓程序停一下，以便記錄當時的ebp(基指針)
Breakpoint 1 at 0x80483ac: file testshellcode.c, line 28.
(gdb) break func #設置斷點在函數入口，以便逐步記錄棧信息
Breakpoint 2 at 0x804835c: file testshellcode.c, line 13.
(gdb) disassemble main   #反編譯main函數，以便記錄調用func後的下一條指令地址
Dump of assembler code for function main:
0x0804839b <main+0>:    lea    0x4(%esp),%ecx
0x0804839f <main+4>:    and    $0xfffffff0,%esp
0x080483a2 <main+7>:    pushl  0xfffffffc(%ecx)
0x080483a5 <main+10>:   push   %ebp
0x080483a6 <main+11>:   mov    %esp,%ebp
0x080483a8 <main+13>:   push   %ecx
0x080483a9 <main+14>:   sub    $0x14,%esp
0x080483ac <main+17>:   push   $0x3
0x080483ae <main+19>:   push   $0x2
0x080483b0 <main+21>:   push   $0x1
0x080483b2 <main+23>:   call   0x8048354 <func>
0x080483b7 <main+28>:   add    $0xc,%esp
0x080483ba <main+31>:   mov    %eax,0xfffffff8(%ebp)
0x080483bd <main+34>:   sub    $0x8,%esp
0x080483c0 <main+37>:   pushl  0xfffffff8(%ebp)
0x080483c3 <main+40>:   push   $0x80484c0
0x080483c8 <main+45>:   call   0x80482a0 <printf@plt>
0x080483cd <main+50>:   add    $0x10,%esp
0x080483d0 <main+53>:   mov    $0x0,%eax
0x080483d5 <main+58>:   mov    0xfffffffc(%ebp),%ecx
0x080483d8 <main+61>:   leave
0x080483d9 <main+62>:   lea    0xfffffffc(%ecx),%esp
0x080483dc <main+65>:   ret
End of assembler dump.
(gdb) r        #運行程序
Starting program: /mnt/hda8/Temp/c/program/testshellcode

Breakpoint 1, main () at testshellcode.c:28
28              sum = func(1, 2, 3);
(gdb) print $ebp  #打印調用func函數之前的基地址，即Previous frame pointer。
$1 = (void *) 0xbf84fdd8
(gdb) n           #執行call指令並跳轉到func函數的入口

Breakpoint 2, func (a=1, b=2, c=3) at testshellcode.c:13
13              int sum = 0;
(gdb) n
16              sum = a + b + c;
(gdb) x/11x $esp  #打印當前棧的內容，可以看出，地址從低到高，注意標記有藍色和紅色的值
                 #它們分別是前一個棧基地址(ebp)和call調用之後的下一條指令的指針(eip)
0xbf84fd94:     0x00000000      0x00000000      0x080482e0      0x00000000
0xbf84fda4:     0xb7f2bce0      0x00000000      0xbf84fdd8      0x080483b7
0xbf84fdb4:     0x00000001      0x00000002      0x00000003
(gdb) n       #執行sum = a + b + c，後，比較棧內容第一行，第4列，由0變為6
18              memset(buffer, '\0', BUF_SIZE);
(gdb) x/11x $esp
0xbf84fd94:     0x00000000      0x00000000      0x080482e0      0x00000006
0xbf84fda4:     0xb7f2bce0      0x00000000      0xbf84fdd8      0x080483b7
0xbf84fdb4:     0x00000001      0x00000002      0x00000003
(gdb) n
19              memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);
(gdb) x/11x $esp #緩衝區初始化以後變成了0
0xbf84fd94:     0x00000000      0x00000000      0x00000000      0x00000006
0xbf84fda4:     0xb7f2bce0      0x00000000      0xbf84fdd8      0x080483b7
0xbf84fdb4:     0x00000001      0x00000002      0x00000003
(gdb) n
21              return sum;
(gdb) x/11x $esp #進行copy以後，這兩列的值變了，大小剛好是7個字節，最後一個字節為'\0'
0xbf84fd94:     0x00000000      0x41414141      0x00414141      0x00000006
0xbf84fda4:     0xb7f2bce0      0x00000000      0xbf84fdd8      0x080483b7
0xbf84fdb4:     0x00000001      0x00000002      0x00000003
(gdb) c
Continuing.
sum = 6

Program exited normally.
(gdb) quit
```

從上面的操作過程，我們可以得出大概的棧分佈(`func` 函數結束之前)如下：


|地址        | 值(hex)   |     符號或者寄存器|   註釋                           |
|------------|-----------|-------------------|----------------------------------|
|低地址      |           |                   |  棧頂方向                        |
|0xbf84fd98  |0x41414141 |    buf[0]         |  可以看出little endian(小端，重要的數據在前面) |
|0xbf84fd9c  |0x00414141 |    buf[1]         |                                                |
|0xbf84fda0  |0x00000006 |    sum            |  可見這上面都是func函數裡頭的局部變量          |
|0xbf84fda4  |0xb7f2bce0 |    esi            |  源索引指針，可以通過產生中間代碼查看，貌似沒什麼作用 |
|0xbf84fda8  |0x00000000 |    edi            |  目的索引指針                                         |
|0xbf84fdac  |0xbf84fdd8 |    ebp            |  調用func之前的棧的基地址，以便調用函數結束之後恢復   |
|0xbf84fdb0  |0x080483b7 |    eip            |  調用func之前的指令指針，以便調用函數結束之後繼續執行 |
|0xbf84fdb4  |0x00000001 |    a              |  第一個參數                                |
|0xbf84fdb8  |0x00000002 |    b              |  第二個參數                                |
|0xbf84fdbc  |0x00000003 |    c              |  第三個參數，可見參數是從最後一個開始壓棧的|
|高地址      |           |                   |  棧底方向                                  |


先說明一下 `edi` 和 `esi` 的由來（在上面的調試過程中我們並沒有看到），是通過產生中間彙編代碼分析得出的。

```
$ gcc -S testshellcode.c
```

在產生的 `testShellcode.s` 代碼裡頭的 `func` 部分看到 `push ebp` 之後就 `push` 了 `edi` 和 `esi` 。但是搜索了一下代碼，發現就這個函數裡頭引用了這兩個寄存器，所以保存它們沒什麼用，刪除以後編譯產生目標代碼後證明是沒用的。

```
$ cat testshellcode.s
...
func:
        pushl   %ebp
        movl    %esp, %ebp
        pushl   %edi
        pushl   %esi
...
        popl    %esi
        popl    %edi
        popl    %ebp
...
```

下面就不管這兩部分（`edi` 和 `esi`）了，主要來分析和函數相關的這幾部分在棧內的分佈：

- 函數局部變量，在靠近棧頂一端
- 調用函數之前的棧的基地址（`ebp`，`Previous Frame Pointer`），在中間靠近棧頂方向
- 調用函數指令的下一條指令地址 ` ` （`eip`），在中間靠近棧底的方向
- 函數參數，在靠近棧底的一端，最後一個參數最先入棧

到這裡，函數調用時的相關內容在棧內的分佈就比較清楚了，在具體分析緩衝區溢出問題之前，我們再來看一個和函數關係很大的問題，即函數返回值的存儲問題：函數的返回值存放在寄存器 `eax` 中。

先來看這段代碼：

```
/**
 * test_return.c -- the return of a function is stored in register eax
 */

#include <stdio.h>

int func()
{
        __asm__ ("movl $1, %eax");
}

int main()
{
        printf("the return of func: %d\n", func());

        return 0;
}

```


編譯運行後，可以看到返回值為 1，剛好是我們在 `func` 函數中 `mov` 到 `eax` 中的“立即數” 1，因此很容易理解返回值存儲在 `eax` 中的事實，如果還有疑慮，可以再看看彙編代碼。在函數返回之後，`eax` 中的值當作了 `printf` 的參數壓入了棧中，而在源代碼中我們正是把 `func` 的結果作為 `printf` 的第二個參數的。

```
$ make test_return
cc     test_return.c   -o test_return
$ ./test_return
the return of func: 1
$ gcc -S test_return.c
$ cat test_return.s
...
        call    func
        subl    $8, %esp
        pushl   %eax      #printf的第二個參數，把func的返回值壓入了棧底
        pushl   $.LC0     #printf的第一個參數the return of func: %d\n
        call    printf
...
```

對於系統調用，返回值也存儲在 `eax` 寄存器中。

<span id="toc_14869_27504_10"></span>
## 緩衝區溢出

<span id="toc_14869_27504_11"></span>
### 實例分析：字符串複製

先來看一段簡短的代碼。

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h>     /* memset, memcpy */

#define BUF_SIZE 8

#ifdef STR1
# define STR_SRC "AAAAAAA\0\1\0\0\0"
#endif

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

int func(int a, int b, int c)
{
        int sum = 0;
        char buffer[BUF_SIZE];

        sum = a + b + c;

        memset(buffer, '\0', BUF_SIZE);
        memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

        return sum;
}

int main()
{
        int sum;

        sum = func(1, 2, 3);

        printf("sum = %d\n", sum);

        return 0;
}
```

編譯一下看看結果：

```
$ gcc -DSTR1 -o testshellcode testshellcode.c  #通過-D定義宏STR1，從而採用第一個STR_SRC的值
$ ./testshellcode
sum = 1
```

不知道你有沒有發現異常呢？上面用紅色標記的地方，本來 `sum` 為 `1+2+3` 即 6，但是實際返回的竟然是 1 。到底是什麼原因呢？大家應該有所瞭解了，因為我們在複製字符串 `AAAAAAA\\0\\1\\0\\0\\0` 到 `buf` 的時候超出 `buf` 本來的大小。 `buf` 本來的大小是 `BUF_SIZE`，8 個字節，而我們要複製的內容是 12 個字節，所以超出了四個字節。根據第一小節的分析，我們用棧的變化情況來表示一下這個複製過程（即執行 `memcpy` 的過程）。

```
memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

（低地址）
複製之前     ====> 複製之後
0x00000000       0x41414141      #char buf[8]
0x00000000       0x00414141
0x00000006       0x00000001      #int sum
（高地址）
```

下面通過 `gdb` 調試來確認一下(只摘錄了一些片斷)。

```
$ gcc -DSTR1 -g -o testshellcode testshellcode.c
$ gdb ./testshellcode
...
(gdb) l
21
22              memset(buffer, '\0', BUF_SIZE);
23              memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);
24
25              return sum;
...
(gdb) break 23
Breakpoint 1 at 0x804837f: file testshellcode.c, line 23.
(gdb) break 25
Breakpoint 2 at 0x8048393: file testshellcode.c, line 25.
(gdb) r
Starting program: /mnt/hda8/Temp/c/program/testshellcode

Breakpoint 1, func (a=1, b=2, c=3) at testshellcode.c:23
23              memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);
(gdb) x/3x $esp+4
0xbfec6bd8:     0x00000000      0x00000000      0x00000006
(gdb) n

Breakpoint 2, func (a=1, b=2, c=3) at testshellcode.c:25
25              return sum;
(gdb) x/3x $esp+4
0xbfec6bd8:     0x41414141      0x00414141      0x00000001
```

可以看出，因為 C 語言沒有對數組的邊界進行限制。我們可以往數組中存入預定義長度的字符串，從而導致緩衝區溢出。

<span id="toc_14869_27504_12"></span>
### 緩衝區溢出後果

溢出之後的問題是導致覆蓋棧的其他內容，從而可能改變程序原來的行為。

如果這類問題被“黑客”利用那將產生非常可怕的後果，小則讓非法用戶獲取了系統權限，把你的服務器當成“殭屍”，用來對其他機器進行攻擊，嚴重的則可能被人刪除數據（所以備份很重要）。即使不被黑客利用，這類問題如果放在醫療領域，那將非常危險，可能那個被覆蓋的數字剛好是用來控制治療癌症的輻射量的，一旦出錯，那可能導致置人死地，當然，如果在航天領域，那可能就是好多個 0 的 `money` 甚至航天員的損失，呵呵，“緩衝區溢出，後果很嚴重！”

<span id="toc_14869_27504_13"></span>
### 緩衝區溢出應對策略

那這個怎麼辦呢？貌似[Linux下緩衝區溢出攻擊的原理及對策](http://www.ibm.com/developerworks/cn/linux/l-overflow/index.html)提到有一個 `libsafe` 庫，可以至少用來檢測程序中出現的類似超出數組邊界的問題。對於上面那個具體問題，為了保護 `sum` 不被修改，有一個小技巧，可以讓求和操作在字符串複製操作之後來做，以便求和操作把溢出的部分給重寫。這個呆夥在下面一塊看效果吧。繼續看看緩衝區的溢出吧。

先來看看這個代碼，還是 `testShellcode.c` 的改進。

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h> /* memset, memcpy */
#define BUF_SIZE 8

#ifdef STR1
# define STR_SRC "AAAAAAAa\1\0\0\0"
#endif
#ifdef STR2
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBB"
#endif
#ifdef STR3
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCC"
#endif
#ifdef STR4
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCCDDDD"
#endif

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

int func(int a, int b, int c)
{
        int sum = 0;
        char buffer[BUF_SIZE] = "";

        memset(buffer, '\0', BUF_SIZE);
        memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

        sum = a + b + c;     //把求和操作放在複製操作之後可以在一定情況下“保護”求和結果

        return sum;
}

int main()
{
        int sum;

        sum = func(1, 2, 3);

        printf("sum = %d\n", sum);

        return 0;
}
```

看看運行情況：

```
$ gcc -D STR2 -o testshellcode testshellcode.c   #再多複製8個字節，結果和STR1時一樣
                        #原因是edi,esi這兩個沒什麼用的，覆蓋了也沒關係
$ ./testshellcode       #看到沒？這種情況下，讓整數操作在字符串複製之後做可以“保護‘整數結果
sum = 6
$ gcc -D STR3 -o testshellcode testshellcode.c  #再多複製4個字節，現在就會把ebp給覆蓋
                                               #了，這樣當main函數再要用ebp訪問數據
                                              #時就會出現訪問非法內存而導致段錯誤。
$ ./testshellcode
Segmentation fault
```
如果感興趣，自己還可以用gdb類似之前一樣來查看複製字符串以後棧的變化情況。

<span id="toc_14869_27504_14"></span>
### 如何保護 ebp 不被修改

下面來做一個比較有趣的事情：如何設法保護我們的 `ebp` 不被修改。

首先要明確 `ebp` 這個寄存器的作用和“行為”，它是棧基地址，並且發現在調用任何一個函數時，這個 `ebp` 總是在第一條指令被壓入棧中，並在最後一條指令（`ret`）之前被彈出。類似這樣：

```
func:                        #函數
       pushl %ebp            #第一條指令
       ...
       popl %ebp             #倒數第二條指令
       ret
```

還記得之前（第一部分）提到的函數的返回值是存儲在 `eax` 寄存器中的麼？如果我們在一個函數中僅僅做放這兩條指令：

```
popl %eax
pushl %eax
```

那不就剛好有：

```
func:                        #函數
       pushl %ebp            #第一條指令
       popl %eax             #把剛壓入棧中的ebp彈出存放到eax中
       pushl %eax            #又把ebp壓入棧
       popl %ebp             #倒數第二條指令
       ret
```


這樣我們沒有改變棧的狀態，卻獲得了 `ebp` 的值，如果在調用任何一個函數之前，獲取這個 `ebp`，並且在任何一條字符串複製語句（可能導致緩衝區溢出的語句）之後重新設置一下 `ebp` 的值，那麼就可以保護 `ebp` 啦。具體怎麼實現呢？看這個代碼。

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h> /* memset, memcpy */
#define BUF_SIZE 8

#ifdef STR1
# define STR_SRC "AAAAAAAa\1\0\0\0"
#endif
#ifdef STR2
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBB"
#endif
#ifdef STR3
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCC"
#endif
#ifdef STR4
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCCDDDD"
#endif

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

unsigned long get_ebp()
{
        __asm__ ("popl %eax;"
                                "pushl %eax;");
}

int func(int a, int b, int c, unsigned long ebp)
{
        int sum = 0;
        char buffer[BUF_SIZE] = "";

        sum = a + b + c;
        memset(buffer, '\0', BUF_SIZE);
        memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);
        *(unsigned long *)(buffer+20) = ebp;
        return sum;
}

int main()
{
        int sum, ebp;

        ebp = get_ebp();
        sum = func(1, 2, 3, ebp);

        printf("sum = %d\n", sum);

        return 0;
}
```

這段代碼和之前的代碼的不同有：

- 給 `func` 函數增加了一個參數 `ebp`，（其實可以用全局變量替代的）
- 利用了剛介紹的原理定義了一個函數 `get_ebp` 以便獲取老的 `ebp` 
- 在 `main` 函數中調用 `func` 之前調用了 `get_ebp`，並把它作為 `func` 的最後一個參數
- 在 `func` 函數中調用 `memcpy` 函數（可能發生緩衝區溢出的地方）之後添加了一條恢復設置 `ebp` 的語句，這條語句先把 `buffer+20` 這個地址（存放 `ebp` 的地址，你可以類似第一部分提到的用 `gdb` 來查看）強制轉換為指向一個 `unsigned long` 型的整數（4 個字節），然後把它指向的內容修改為老的 `ebp` 。

看看效果：

```
$ gcc -D STR3 -o testshellcode testshellcode.c
$ ./testshellcode         #現在沒有段錯誤了吧，因為ebp得到了“保護”
sum = 6
```

<span id="toc_14869_27504_15"></span>
### 如何保護 eip 不被修改？

如果我們複製更多的字節過去了，比如再多複製四個字節進去，那麼 `eip` 就被覆蓋了。

```
$ gcc -D STR4 -o testshellcode testshellcode.c
$ ./testshellcode
Segmentation fault
```

同樣會出現段錯誤，因為下一條指令的位置都被改寫了，`func` 返回後都不知道要訪問哪個”非法“地址啦。呵呵，如果是一個合法地址呢？

如果在緩衝區溢出時，`eip` 被覆蓋了，並且被修改為了一條合法地址，那麼問題就非常”有趣“了。如果這個地址剛好是調用func的那個地址，那麼整個程序就成了死循環，如果這個地址指向的位置剛好有一段關機代碼，那麼系統正在運行的所有服務都將被關掉，如果那個地方是一段更惡意的代碼，那就？你可以盡情想像哦。如果是黑客故意利用這個，那麼那些代碼貌似就叫做[shellcode](http://janxin.bokee.com/4067220.html)了。

有沒有保護 `eip` 的辦法呢？呵呵，應該是有的吧。不知道 `gas` 有沒有類似 `masm` 彙編器中 `offset` 的偽操作指令（查找了一下，貌似沒有），如果有的話在函數調用之前設置一個標號，在後面某個位置獲取，再加上一個可能的偏移（包括 `call` 指令的長度和一些 `push` 指令等），應該可以算出來，不過貌似比較麻煩（或許你靈感大作，找到好辦法了！），這裡直接通過 `gdb` 反彙編求得它相對 `main` 的偏移算出來得了。求出來以後用它來”保護“棧中的值。

看看這個代碼：

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h> /* memset, memcpy */
#define BUF_SIZE 8

#ifdef STR1
# define STR_SRC "AAAAAAAa\1\0\0\0"
#endif
#ifdef STR2
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBB"
#endif
#ifdef STR3
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCC"
#endif
#ifdef STR4
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCCDDDD"
#endif

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

int main();
#define OFFSET  40

unsigned long get_ebp()
{
        __asm__ ("popl %eax;"
                                "pushl %eax;");
}

int func(int a, int b, int c, unsigned long ebp)
{
        int sum = 0;
        char buffer[BUF_SIZE] = "";

        memset(buffer, '\0', BUF_SIZE);
        memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

        sum = a + b + c;

        *(unsigned long *)(buffer+20) = ebp;
        *(unsigned long *)(buffer+24) = (unsigned long)main+OFFSET;
        return sum;
}

int main()
{
        int sum, ebp;

        ebp = get_ebp();
        sum = func(1, 2, 3, ebp);

        printf("sum = %d\n", sum);

        return 0;
}
```

看看效果：

```
$ gcc -D STR4 -o testshellcode testshellcode.c
$ ./testshellcode
sum = 6
```

這樣，`EIP` 也得到了“保護”（這個方法很糟糕的，呵呵）。

類似地，如果再多複製一些內容呢？那麼棧後面的內容都將被覆蓋，即傳遞給 `func` 函數的參數都將被覆蓋，因此上面的方法，包括所謂的對 `sum` 和 `ebp` 等值的保護都沒有任何意義了（如果再對後面的參數進行進一步的保護呢？或許有點意義，呵呵）。在這裡，之所以提出類似這樣的保護方法，實際上只是為了討論一些有趣的細節並加深對緩衝區溢出這一問題的理解（或許有一些實際的價值哦，算是拋磚引玉吧）。

<span id="toc_14869_27504_16"></span>
### 緩衝區溢出檢測

要確實解決這類問題，從主觀上講，還得程序員來做相關的工作，比如限制將要複製的字符串的長度，保證它不超過當初申請的緩衝區的大小。

例如，在上面的代碼中，我們在 `memcpy` 之前，可以加入一個判斷，並且可以對緩衝區溢出進行很好的檢查。如果能夠設計一些比較好的測試實例把這些判斷覆蓋到，那麼相關的問題就可以得到比較不錯的檢查了。

```
/* testshellcode.c */
#include <stdio.h>      /* printf */
#include <string.h> /* memset, memcpy */
#include <stdlib.h>     /* exit */
#define BUF_SIZE 8

#ifdef STR4
# define STR_SRC "AAAAAAAa\1\0\0\0BBBBBBBBCCCCDDDD"
#endif

#ifndef STR_SRC
# define STR_SRC "AAAAAAA"
#endif

int func(int a, int b, int c)
{
        int sum = 0;
        char buffer[BUF_SIZE] = "";

        memset(buffer, '\0', BUF_SIZE);
        if ( sizeof(STR_SRC)-1 > BUF_SIZE ) {
                printf("buffer overflow!\n");
                exit(-1);
        }
        memcpy(buffer, STR_SRC, sizeof(STR_SRC)-1);

        sum = a + b + c;

        return sum;
}

int main()
{
        int sum;

        sum = func(1, 2, 3);

        printf("sum = %d\n", sum);

        return 0;
}
```

現在的效果如下：

```
$ gcc -DSTR4 -g -o testshellcode testshellcode.c
$ ./testshellcode      #如果存在溢出，那麼就會得到阻止並退出，從而阻止可能的破壞
buffer overflow!
$ gcc -g -o testshellcode testshellcode.c
$ ./testshellcode
sum = 6
```

當然，如果能夠在 C 標準裡頭加入對數組操作的限制可能會更好，或者在編譯器中擴展對可能引起緩衝區溢出的語法檢查。

<span id="toc_14869_27504_17"></span>
## 緩衝區注入實例

最後給出一個利用上述緩衝區溢出來進行緩衝區注入的例子。也就是通過往某個緩衝區注入一些代碼，並把eip修改為這些代碼的入口從而達到破壞目標程序行為的目的。

這個例子來自[Linux 下緩衝區溢出攻擊的原理及對策](http://www.ibm.com/developerworks/cn/linux/l-overflow/index.html)，這裡主要利用上面介紹的知識對它進行了比較詳細的分析。

<span id="toc_14869_27504_18"></span>
### 準備：把 C 語言函數轉換為字符串序列

首先回到第一部分，看看那個 `Shellcode.c` 程序。我們想獲取它的彙編代碼，並以十六進制字節的形式輸出，以便把這些指令當字符串存放起來，從而作為緩衝區注入時的輸入字符串。下面通過 `gdb` 獲取這些內容。

```
$ gcc -g -o shellcode shellcode.c
$ gdb ./shellcode
GNU gdb 6.6
...
(gdb) disassemble main
Dump of assembler code for function main:
...
0x08048331 <main+13>:   push   %ecx
0x08048332 <main+14>:   jmp    0x8048354 <forward>
0x08048334 <main+16>:   pop    %esi
0x08048335 <main+17>:   mov    $0x4,%eax
0x0804833a <main+22>:   mov    $0x2,%ebx
0x0804833f <main+27>:   mov    %esi,%ecx
0x08048341 <main+29>:   mov    $0xc,%edx
0x08048346 <main+34>:   int    $0x80
0x08048348 <main+36>:   mov    $0x1,%eax
0x0804834d <main+41>:   mov    $0x0,%ebx
0x08048352 <main+46>:   int    $0x80
0x08048354 <forward+0>: call   0x8048334 <main+16>
0x08048359 <forward+5>: dec    %eax
0x0804835a <forward+6>: gs
0x0804835b <forward+7>: insb   (%dx),%es:(%edi)
0x0804835c <forward+8>: insb   (%dx),%es:(%edi)
0x0804835d <forward+9>: outsl  %ds:(%esi),(%dx)
0x0804835e <forward+10>:        and    %dl,0x6f(%edi)
0x08048361 <forward+13>:        jb     0x80483cf <__libc_csu_init+79>
0x08048363 <forward+15>:        or     %fs:(%eax),%al
...
End of assembler dump.
(gdb) set logging on   #開啟日誌功能，記錄操作結果
Copying output to gdb.txt.
(gdb) x/52bx main+14  #以十六進制單字節（字符）方式打印出shellcode的核心代碼
0x8048332 <main+14>:    0xeb    0x20    0x5e    0xb8    0x04    0x00    0x00   0x00
0x804833a <main+22>:    0xbb    0x02    0x00    0x00    0x00    0x89    0xf1   0xba
0x8048342 <main+30>:    0x0c    0x00    0x00    0x00    0xcd    0x80    0xb8   0x01
0x804834a <main+38>:    0x00    0x00    0x00    0xbb    0x00    0x00    0x00   0x00
0x8048352 <main+46>:    0xcd    0x80    0xe8    0xdb    0xff    0xff    0xff   0x48
0x804835a <forward+6>:  0x65    0x6c    0x6c    0x6f    0x20    0x57    0x6f   0x72
0x8048362 <forward+14>: 0x6c    0x64    0x0a    0x00
(gdb) quit
$ cat gdb.txt | sed -e "s/^.*://g;s/\t/\\\/g;s/^/\"/g;s/\$/\"/g"  #把日誌裡頭的內容處理一下，得到這樣一個字符串
"\0xeb\0x20\0x5e\0xb8\0x04\0x00\0x00\0x00"
"\0xbb\0x02\0x00\0x00\0x00\0x89\0xf1\0xba"
"\0x0c\0x00\0x00\0x00\0xcd\0x80\0xb8\0x01"
"\0x00\0x00\0x00\0xbb\0x00\0x00\0x00\0x00"
"\0xcd\0x80\0xe8\0xdb\0xff\0xff\0xff\0x48"
"\0x65\0x6c\0x6c\0x6f\0x20\0x57\0x6f\0x72"
"\0x6c\0x64\0x0a\0x00"
```

<span id="toc_14869_27504_19"></span>
### 注入：在 C 語言中執行字符串化的代碼

得到上面的字符串以後我們就可以設計一段下面的代碼啦。

```
/* testshellcode.c */
char shellcode[]="\xeb\x20\x5e\xb8\x04\x00\x00\x00"
"\xbb\x02\x00\x00\x00\x89\xf1\xba"
"\x0c\x00\x00\x00\xcd\x80\xb8\x01"
"\x00\x00\x00\xbb\x00\x00\x00\x00"
"\xcd\x80\xe8\xdb\xff\xff\xff\x48"
"\x65\x6c\x6c\x6f\x20\x57\x6f\x72"
"\x6c\x64\x0a\x00";

void callshellcode(void)
{
   int *ret;
   ret = (int *)&ret + 2;
   (*ret) = (int)shellcode;
}

int main()
{
        callshellcode();

        return 0;
}
```

運行看看，

```
$ gcc -o testshellcode testshellcode.c
$ ./testshellcode
Hello World
```

竟然打印出了 `Hello World`，實際上，如果只是為了讓 `Shellcode` 執行，有更簡單的辦法，直接把 `Shellcode` 這個字符串入口強制轉換為一個函數入口，並調用就可以，具體見這段代碼。

```
char shellcode[]="\xeb\x20\x5e\xb8\x04\x00\x00\x00"
"\xbb\x02\x00\x00\x00\x89\xf1\xba"
"\x0c\x00\x00\x00\xcd\x80\xb8\x01"
"\x00\x00\x00\xbb\x00\x00\x00\x00"
"\xcd\x80\xe8\xdb\xff\xff\xff\x48"
"\x65\x6c\x6c\x6f\x20\x57\x6f\x72"
"\x6c\x64\x0a\x00";

typedef void (* func)();            //定義一個指向函數的指針func，而函數的返回值和參數均為void

int main()
{
        (* (func)shellcode)();

        return 0;
}
```

<span id="toc_14869_27504_20"></span>
### 注入原理分析

這裡不那樣做，為什麼也能夠執行到 `Shellcode` 呢？仔細分析一下 `callShellcode` 裡頭的代碼就可以得到原因了。

```
int *ret;
```

這裡定義了一個指向整數的指針，`ret` 佔用 4 個字節（可以用 `sizeof(int *)` 算出）。

```
ret = (int *)&ret + 2;
```

這裡把 `ret` 修改為它本身所在的地址再加上兩個單位。
首先需要求出 `ret` 本身所在的位置，因為 `ret` 是函數的一個局部變量，它在棧中偏棧頂的地方。
然後呢？再增加兩個單位，這個單位是 `sizeof(int)`，即 4 個字節。這樣，新的 `ret` 就是 `ret` 所在的位置加上 8 個字節，即往棧底方向偏移 8 個字節的位置。對於我們之前分析的 `Shellcode`，那裡應該是 `edi`，但實際上這裡並不是 `edi`，可能是 `gcc` 在編譯程序時有不同的處理，這裡實際上剛好是 `eip`，即執行這條語句之後 `ret` 的值變成了 `eip` 所在的位置。

```
(*ret) = (int)shellcode;
```

由於之前 `ret` 已經被修改為了 `eip` 所在的位置，這樣對 `(*ret)` 賦值就會修改 `eip` 的值，即下一條指令的地址，這裡把 `eip` 修改為了 `Shellcode` 的入口。因此，當函數返回時直接去執行 `Shellcode` 裡頭的代碼，並打印了 `Hello World` 。

用 `gdb` 調試一下看看相關變量的值的情況。這裡主要關心 `ret` 本身。 `ret` 本身是一個地址，首先它所在的位置變成了 `EIP` 所在的位置（把它自己所在的位置加上 `2*4` 以後賦於自己），然後，`EIP` 又指向了 `Shellcode` 處的代碼。

```
$ gcc -g -o testshellcode testshellcode.c
$ gdb ./testshellcode
...
(gdb) l
8       void callshellcode(void)
9       {
10         int *ret;
11         ret = (int *)&ret + 2;
12         (*ret) = (int)shellcode;
13      }
14
15      int main()
16      {
17              callshellcode();
(gdb) break 17
Breakpoint 1 at 0x804834d: file testshell.c, line 17.
(gdb) break 11
Breakpoint 2 at 0x804832a: file testshell.c, line 11.
(gdb) break 12
Breakpoint 3 at 0x8048333: file testshell.c, line 12.
(gdb) break 13
Breakpoint 4 at 0x804833d: file testshell.c, line 13.
(gdb) r
Starting program: /mnt/hda8/Temp/c/program/testshell

Breakpoint 1, main () at testshell.c:17
17              callshellcode();
(gdb) print $ebp       #打印ebp寄存器裡的值
$1 = (void *) 0xbfcfd2c8
(gdb) disassemble main
...
0x0804834d <main+14>:   call   0x8048324 <callshellcode>
0x08048352 <main+19>:   mov    $0x0,%eax
...
(gdb) n

Breakpoint 2, callshellcode () at testshell.c:11
11         ret = (int *)&ret + 2;
(gdb) x/6x $esp
0xbfcfd2ac:     0x08048389      0xb7f4eff4      0xbfcfd36c      0xbfcfd2d8
0xbfcfd2bc:     0xbfcfd2c8      0x08048352
(gdb) print &ret #分別打印出ret所在的地址和ret的值，剛好在ebp之上，我們發現這裡並沒有
       #之前的testshellcode代碼中的edi和esi，可能是gcc在彙編的時候有不同處理。
$2 = (int **) 0xbfcfd2b8
(gdb) print ret
$3 = (int *) 0xbfcfd2d8 #這裡的ret是個隨機值
(gdb) n

Breakpoint 3, callshellcode () at testshell.c:12
12         (*ret) = (int)shellcode;
(gdb) print ret   #執行完ret = (int *)&ret + 2;後，ret變成了自己地址加上2*4，
                  #剛好是eip所在的位置。
$5 = (int *) 0xbfcfd2c0
(gdb) x/6x $esp
0xbfcfd2ac:     0x08048389      0xb7f4eff4      0xbfcfd36c      0xbfcfd2c0
0xbfcfd2bc:     0xbfcfd2c8      0x08048352
(gdb) x/4x *ret  #此時*ret剛好為eip，0x8048352
0x8048352 <main+19>:    0x000000b8      0x8d5d5900      0x90c3fc61      0x89559090
(gdb) n

Breakpoint 4, callshellcode () at testshell.c:13
13      }
(gdb) x/6x $esp #現在eip被修改為了shellcode的入口
0xbfcfd2ac:     0x08048389      0xb7f4eff4      0xbfcfd36c      0xbfcfd2c0
0xbfcfd2bc:     0xbfcfd2c8      0x8049560
(gdb) x/4x *ret  #現在修改了(*ret)的值，即修改了eip的值，使eip指向了shellcode
0x8049560 <shellcode>:  0xb85e20eb      0x00000004      0x000002bb      0xbaf18900
```

上面的過程很難弄，呵呵。主要是指針不大好理解，如果直接把它當地址繪出下面的圖可能會容易理解一些。

callshellcode棧的初始分佈：

```
ret=(int *)&ret+2=0xbfcfd2bc+2*4=0xbfcfd2c0
0xbfcfd2b8      ret(隨機值)                     0xbfcfd2c0
0xbfcfd2bc      ebp(這裡不關心)
0xbfcfd2c0      eip(0x08048352)         eip(0x8049560 )

(*ret) = (int)shellcode;即eip=0x8049560
```

總之，最後體現為函數調用的下一條指令指針（`eip`）被修改為一段注入代碼的入口，從而使得函數返回時執行了注入代碼。

<span id="toc_14869_27504_21"></span>
### 緩衝區注入與防範

這個程序裡頭的注入代碼和被注入程序竟然是一個程序，傻瓜才自己攻擊自己（不過有些黑客有可能利用程序中一些空閒空間注入代碼哦），真正的緩衝區注入程序是分開的，比如作為被注入程序的一個字符串參數。而在被注入程序中剛好沒有做字符串長度的限制，從而讓這段字符串中的一部分修改了 `eip`，另外一部分作為注入代碼運行了，從而實現了注入的目的。不過這會涉及到一些技巧，即如何剛好用注入代碼的入口地址來修改 `eip` （即新的 `eip` 能夠指向注入代碼）？如果 `eip` 的位置和緩衝區的位置之間的距離是確定，那麼就比較好處理了，但從上面的兩個例子中我們發現，有一個編譯後有 `edi` 和 `esi`，而另外一個則沒有，另外，緩衝區的位置，以及被注入程序有多少個參數我們都無法預知，因此，如何計算 `eip` 所在的位置呢？這也會很難確定。還有，為了防止緩衝區溢出帶來的注入問題，現在的操作系統採取了一些辦法，比如讓 `esp` 隨機變化（比如和系統時鐘關聯起來），所以這些措施將導致注入更加困難。如果有興趣，你可以接著看看最後的幾篇參考資料並進行更深入的研究。

需要提到的是，因為很多程序可能使用 `strcpy 來進行字符串的複製，在實際編寫緩衝區注入代碼時，會採取一定的辦法（指令替換），把代碼中可能包含的 `\\0 字節去掉，從而防止 `strcpy 中斷對注入代碼的複製，進而可以複製完整的注入代碼。具體的技巧可以參考 [Linux下緩衝區溢出攻擊的原理及對策](http://www.ibm.com/developerworks/cn/linux/l-overflow/index.html)，[Shellcode技術雜談](http://janxin.bokee.com/4067220.html)，[virus-writing-HOWTO](http://virus.bartolich.at/virus-writing-HOWTO/_html/)。

<span id="toc_14869_27504_22"></span>
## 後記

實際上緩衝區溢出應該是語法和邏輯方面的雙重問題，由於語法上的不嚴格（對數組邊界沒有檢查）導致邏輯上可能出現嚴重缺陷（程序執行行為被改變）。另外，這類問題是對程序運行過程中的程序映像的棧區進行注入。實際上除此之外，程序在安全方面還有很多類似的問題。比如，雖然程序映像的正文區受到系統保護（只讀），但是如果內存（硬件本身，內存條）出現故障，在程序運行的過程中，程序映像的正文區的某些字節就可能被修改了，也可能發生非常嚴重的後果，因此程序運行過程的正文區檢查等可能的手段需要被引入。

<span id="toc_14869_27504_23"></span>
## 參考資料

-   Playing with ptrace
    - [how ptrace can be used to trace system calls and change system call arguments](http://www.linuxjournal.com/article/6100)
    - [setting breakpoints and injecting code into running programs](http://www.linuxjournal.com/node/6210/print)
    - [fix the problem of ORIG_EAX not defined](http://www.ecos.sourceware.org/ml/libc-hacker/1998-05/msg00277.html)
-   [《緩衝區溢出攻擊—— 檢測、剖析與預防》第五章](http://book.csdn.net/bookfiles/228/index.html)
-   [Linux下緩衝區溢出攻擊的原理及對策](http://www.ibm.com/developerworks/cn/linux/l-overflow/index.html)
-   [Linux 彙編語言開發指南](http://www.ibm.com/developerworks/cn/linux/l-assembly/index.html)
-   [Shellcode 技術雜談](http://janxin.bokee.com/4067220.html)
