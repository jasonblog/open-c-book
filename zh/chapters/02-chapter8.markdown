# 打造史上最小可執行 ELF 文件（45 字節，可打印字符串）

-    [前言](#toc_3928_6176_1)
-    [可執行文件格式的選取](#toc_3928_6176_2)
-    [鏈接優化](#toc_3928_6176_3)
-    [可執行文件“減肥”實例（從6442到708字節）](#toc_3928_6176_4)
    -    [系統默認編譯](#toc_3928_6176_5)
    -    [不採用默認編譯](#toc_3928_6176_6)
    -    [刪除對程序運行沒有影響的節區](#toc_3928_6176_7)
    -    [刪除可執行文件的節區表](#toc_3928_6176_8)
-    [用匯編語言來重寫 Hello World（76字節）](#toc_3928_6176_9)
    -    [採用默認編譯](#toc_3928_6176_10)
    -    [刪除掉彙編代碼中無關緊要內容](#toc_3928_6176_11)
    -    [不默認編譯並刪除掉無關節區和節區表](#toc_3928_6176_12)
    -    [用系統調用取代庫函數](#toc_3928_6176_13)
    -    [把字符串作為參數輸入](#toc_3928_6176_14)
    -    [寄存器賦值重用](#toc_3928_6176_15)
    -    [通過文件名傳遞參數](#toc_3928_6176_16)
    -    [刪除非必要指令](#toc_3928_6176_17)
-    [合併代碼段、程序頭和文件頭（52字節）](#toc_3928_6176_18)
    -    [把代碼段移入文件頭](#toc_3928_6176_19)
    -    [把程序頭移入文件頭](#toc_3928_6176_20)
    -    [在非連續的空間插入代碼](#toc_3928_6176_21)
    -    [把程序頭完全合入文件頭](#toc_3928_6176_22)
-    [彙編語言極限精簡之道（45字節）](#toc_3928_6176_23)
-    [小結](#toc_3928_6176_24)
-    [參考資料](#toc_3928_6176_25)


<span id="toc_3928_6176_1"></span>
## 前言

本文從減少可執行文件大小的角度分析了 `ELF` 文件，期間通過經典的 `Hello World` 實例逐步演示如何通過各種常用工具來分析 `ELF` 文件，並逐步精簡代碼。

為了能夠儘量減少可執行文件的大小，我們必須瞭解可執行文件的格式，以及鏈接生成可執行文件時的後臺細節（即最終到底有哪些內容被鏈接到了目標代碼中）。通過選擇合適的可執行文件格式並剔除對可執行文件的最終運行沒有影響的內容，就可以實現目標代碼的裁減。因此，通過探索減少可執行文件大小的方法，就相當於實踐性地去探索了可執行文件的格式以及鏈接過程的細節。

當然，算法的優化和編程語言的選擇可能對目標文件的大小有很大的影響，在本文最後我們會跟參考資料 [\[1\]][1] 的作者那樣去探求一個打印 `Hello World` 的可執行文件能夠小到什麼樣的地步。

<span id="toc_3928_6176_2"></span>
## 可執行文件格式的選取

可執行文件格式的選擇要滿足的一個基本條件是：目標系統支持該可執行文件格式，資料 [\[2\]][2] 分析和比較了 `UNIX` 平臺下的三種可執行文件格式，這三種格式實際上代表著可執行文件的一個發展過程：

- a.out 文件格式非常緊湊，只包含了程序運行所必須的信息（文本、數據、 `BSS`），而且每個 `section` 的順序是固定的。

- coff 文件格式雖然引入了一個節區表以支持更多節區信息，從而提高了可擴展性，但是這種文件格式的重定位在鏈接時就已經完成，因此不支持動態鏈接（不過擴展的 `coff` 支持）。

- elf 文件格式不僅動態鏈接，而且有很好的擴展性。它可以描述可重定位文件、可執行文件和可共享文件（動態鏈接庫）三類文件。

下面來看看 `ELF` 文件的結構圖：

```
文件頭部(ELF Header)
程序頭部表(Program Header Table)
節區1(Section1)
節區2(Section2)
節區3(Section3)
...
節區頭部(Section Header Table)
```

無論是文件頭部、程序頭部表、節區頭部表還是各個節區，都是通過特定的結構體 `(struct)描述的，這些結構在 `elf.h` 文件中定義。文件頭部用於描述整個文件的類型、大小、運行平臺、程序入口、程序頭部表和節區頭部表等信息。例如，我們可以通過文件頭部查看該 `ELF` 文件的類型。

```
$ cat hello.c   #典型的hello, world程序
#include <stdio.h>

int main(void)
{
	printf("hello, world!\n");
	return 0;
}
$ gcc -c hello.c   #編譯，產生可重定向的目標代碼
$ readelf -h hello.o | grep Type   #通過readelf查看文件頭部找出該類型
  Type:                              REL (Relocatable file)
$ gcc -o hello hello.o   #生成可執行文件
$ readelf -h hello | grep Type
  Type:                              EXEC (Executable file)
$ gcc -fpic -shared -W1,-soname,libhello.so.0 -o libhello.so.0.0 hello.o  #生成共享庫
$ readelf -h libhello.so.0.0 | grep Type
  Type:                              DYN (Shared object file)
```

那節區頭部表（將簡稱節區表）和程序頭部表有什麼用呢？實際上前者只對可重定向文件有用，而後者只對可執行文件和可共享文件有用。

節區表是用來描述各節區的，包括各節區的名字、大小、類型、虛擬內存中的位置、相對文件頭的位置等，這樣所有節區都通過節區表給描述了，這樣連接器就可以根據文件頭部表和節區表的描述信息對各種輸入的可重定位文件進行合適的鏈接，包括節區的合併與重組、符號的重定位（確認符號在虛擬內存中的地址）等，把各個可重定向輸入文件鏈接成一個可執行文件（或者是可共享文件）。如果可執行文件中使用了動態連接庫，那麼將包含一些用於動態符號鏈接的節區。我們可以通過 `readelf -S` （或 `objdump -h`）查看節區表信息。

```
$ readelf -S hello  #可執行文件、可共享庫、可重定位文件默認都生成有節區表
...
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        08048114 000114 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048128 000128 000020 00   A  0   0  4
  [ 3] .hash             HASH            08048148 000148 000028 04   A  5   0  4
...
    [ 7] .gnu.version      VERSYM          0804822a 00022a 00000a 02   A  5   0  2
...
  [11] .init             PROGBITS        08048274 000274 000030 00  AX  0   0  4
...
  [13] .text             PROGBITS        080482f0 0002f0 000148 00  AX  0   0 16
  [14] .fini             PROGBITS        08048438 000438 00001c 00  AX  0   0  4
...
```

三種類型文件的節區（各個常見節區的作用請參考資料 [\[11\]][11])可能不一樣，但是有幾個節區，例如 `.text`，`.data`，`.bss` 是必須的，特別是 `.text`，因為這個節區包含了代碼。如果一個程序使用了動態鏈接庫（引用了動態連接庫中的某個函數），那麼需要 `.interp` 節區以便告知系統使用什麼動態連接器程序來進行動態符號鏈接，進行某些符號地址的重定位。通常，`.rel.text` 節區只有可重定向文件有，用於鏈接時對代碼區進行重定向，而 `.hash`，`.plt`，`.got` 等節區則只有可執行文件（或可共享庫）有，這些節區對程序的運行特別重要。還有一些節區，可能僅僅是用於註釋，比如 `.comment`，這些對程序的運行似乎沒有影響，是可有可無的，不過有些節區雖然對程序的運行沒有用處，但是卻可以用來輔助對程序進行調試或者對程序運行效率有影響。

雖然三類文件都必須包含某些節區，但是節區表對可重定位文件來說才是必須的，而程序的執行卻不需要節區表，只需要程序頭部表以便知道如何加載和執行文件。不過如果需要對可執行文件或者動態連接庫進行調試，那麼節區表卻是必要的，否則調試器將不知道如何工作。下面來介紹程序頭部表，它可通過 `readelf -l`（或 `objdump -p`）查看。

```
$ readelf -l hello.o #對於可重定向文件，gcc沒有產生程序頭部，因為它對可重定向文件沒用

There are no program headers in this file.
$  readelf -l hello  #而可執行文件和可共享文件都有程序頭部
...
Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x000e0 0x000e0 R E 0x4
  INTERP         0x000114 0x08048114 0x08048114 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x00470 0x00470 R E 0x1000
  LOAD           0x000470 0x08049470 0x08049470 0x0010c 0x00110 RW  0x1000
  DYNAMIC        0x000484 0x08049484 0x08049484 0x000d0 0x000d0 RW  0x4
  NOTE           0x000128 0x08048128 0x08048128 0x00020 0x00020 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame
   03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag
   06
$ readelf -l libhello.so.0.0  #節區和上面類似，這裡省略
```

從上面可看出程序頭部表描述了一些段（`Segment`），這些段對應著一個或者多個節區，上面的 `readelf -l` 很好地顯示了各個段與節區的映射。這些段描述了段的名字、類型、大小、第一個字節在文件中的位置、將佔用的虛擬內存大小、在虛擬內存中的位置等。這樣系統程序解釋器將知道如何把可執行文件加載到內存中以及進行動態鏈接等動作。

該可執行文件包含 7 個段，`PHDR` 指程序頭部，`INTERP` 正好對應 `.interp` 節區，兩個 `LOAD` 段包含程序的代碼和數據部分，分別包含有 `.text` 和 `.data`，`.bss` 節區，`DYNAMIC` 段包含 `.daynamic`，這個節區可能包含動態連接庫的搜索路徑、可重定位表的地址等信息，它們用於動態連接器。 `NOTE` 和 `GNU_STACK` 段貌似作用不大，只是保存了一些輔助信息。因此，對於一個不使用動態連接庫的程序來說，可能只包含 `LOAD` 段，如果一個程序沒有數據，那麼只有一個 `LOAD` 段就可以了。

總結一下，Linux 雖然支持很多種可執行文件格式，但是目前 `ELF` 較通用，所以選擇 `ELF` 作為我們的討論對象。通過上面對 `ELF` 文件分析發現一個可執行的文件可能包含一些對它的運行沒用的信息，比如節區表、一些用於調試、註釋的節區。如果能夠刪除這些信息就可以減少可執行文件的大小，而且不會影響可執行文件的正常運行。

<span id="toc_3928_6176_3"></span>
## 鏈接優化

從上面的討論中已經接觸了動態連接庫。 `ELF` 中引入動態連接庫後極大地方便了公共函數的共享，節約了磁盤和內存空間，因為不再需要把那些公共函數的代碼鏈接到可執行文件，這將減少了可執行文件的大小。

與此同時，靜態鏈接可能會引入一些對代碼的運行可能並非必須的內容。你可以從[《GCC 編譯的背後（第二部分：彙編和鏈接）》][100] 瞭解到 `GCC` 鏈接的細節。從那篇 Blog 中似乎可以得出這樣的結論：僅僅從是否影響一個 C 語言程序運行的角度上說，`GCC` 默認鏈接到可執行文件的幾個可重定位文件 （`crt1.o`，`rti.o`，`crtbegin.o`，`crtend.o`，`crtn.o`）並不是必須的，不過值得注意的是，如果沒有鏈接那些文件但在程序末尾使用了 `return` 語句，`main` 函數將無法返回，因此需要替換為 `_exit` 調用；另外，既然程序在進入 `main` 之前有一個入口，那麼 `main` 入口就不是必須的。因此，如果不採用默認鏈接也可以減少可執行文件的大小。

[100]: 02-chapter2.markdown

<span id="toc_3928_6176_4"></span>
## 可執行文件“減肥”實例（從6442到708字節）

這裡主要是根據上面兩點來介紹如何減少一個可執行文件的大小。以 `Hello World` 為例。

首先來看看默認編譯產生的 `Hello World` 的可執行文件大小。

<span id="toc_3928_6176_5"></span>
### 系統默認編譯

代碼同上，下面是一組演示，

```
$ uname -r   #先查看內核版本和gcc版本，以便和你的結果比較
2.6.22-14-generic
$ gcc --version
gcc (GCC) 4.1.3 20070929 (prerelease) (Ubuntu 4.1.2-16ubuntu2)
...
$ gcc -o hello hello.c   #默認編譯
$ wc -c hello   #產生一個大小為6442字節的可執行文件
6442 hello
```

<span id="toc_3928_6176_6"></span>
### 不採用默認編譯

可以考慮編輯時就把 `return 0` 替換成 `_exit(0)` 幷包含定義該函數的 `unistd.h` 頭文件。下面是從[《GCC 編譯的背後（第二部分：彙編和鏈接）》][100]總結出的 `Makefile` 文件。

[100]: 02-chapter2.markdown

```
#file: Makefile
#functin: for not linking a program as the gcc do by default
#author: falcon<zhangjinw@gmail.com>
#update: 2008-02-23

MAIN = hello
SOURCE =
OBJS = hello.o
TARGET = hello
CC = gcc-3.4 -m32
LD = ld -m elf_i386

CFLAGSs += -S
CFLAGSc += -c
LDFLAGS += -dynamic-linker /lib/ld-linux.so.2 -L /usr/lib/ -L /lib -lc
RM = rm -f
SEDc = sed -i -e '/\#include[ "<]*unistd.h[ ">]*/d;' \
	-i -e '1i \#include <unistd.h>' \
	-i -e 's/return 0;/_exit(0);/'
SEDs = sed -i -e 's/main/_start/g'

all: $(TARGET)

$(TARGET):
	@$(SEDc) $(MAIN).c
	@$(CC) $(CFLAGSs) $(MAIN).c
	@$(SEDs) $(MAIN).s
	@$(CC) $(CFLAGSc) $(MAIN).s $(SOURCE)
	@$(LD) $(LDFLAGS) -o $@ $(OBJS)
clean:
	@$(RM) $(MAIN).s $(OBJS) $(TARGET)
```

把上面的代碼複製到一個Makefile文件中，並利用它來編譯hello.c。

```
$ make   #編譯
$ ./hello   #這個也是可以正常工作的
Hello World
$ wc -c hello   #但是大小減少了4382個字節，減少了將近 70%
2060 hello
$ echo "6442-2060" | bc
4382
$ echo "(6442-2060)/6442" | bc -l
.68022353306426575597
```

對於一個比較小的程序，能夠減少將近 70% “沒用的”代碼。

<span id="toc_3928_6176_7"></span>
### 刪除對程序運行沒有影響的節區

使用上述 `Makefile` 來編譯程序，不鏈接那些對程序運行沒有多大影響的文件，實際上也相當於刪除了一些“沒用”的節區，可以通過下列演示看出這個實質。

```
$ make clean
$ make
$ readelf -l hello | grep "0[0-9]\ \ "
   00
   01     .interp
   02     .interp .hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.plt .plt .text .rodata
   03     .dynamic .got.plt
   04     .dynamic
   05
$ make clean
$ gcc -o hello hello.c
$ readelf -l hello | grep "0[0-9]\ \ "
   00
   01     .interp
   02     .interp .note.ABI-tag .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r
	  .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame
   03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss
   04     .dynamic
   05     .note.ABI-tag
   06
```

通過比較發現使用自定義的 `Makefile` 文件，少了這麼多節區： `.bss .ctors .data .dtors .eh_frame .fini .gnu.hash .got .init .jcr .note.ABI-tag .rel.dyn` 。
再看看還有哪些節區可以刪除呢？通過之前的分析發現有些節區是必須的，那 `.hash?.gnu.version?` 呢，通過 `strip -R`（或 `objcop -R`）刪除這些節區試試。

```
$ wc -c hello   #查看大小，以便比較
2060
$ time ./hello    #我們比較一下一些節區對執行時間可能存在的影響
Hello World

real    0m0.001s
user    0m0.000s
sys     0m0.000s
$ strip -R .hash hello   #刪除.hash節區
$ wc -c hello
1448 hello
$ echo "2060-1448" | bc   #減少了612字節
612
$ time ./hello           #發現執行時間長了一些（實際上也可能是進程調度的問題）
Hello World

real    0m0.006s
user    0m0.000s
sys     0m0.000s
$ strip -R .gnu.version hello   #刪除.gnu.version還是可以工作
$ wc -c hello
1396 hello
$ echo "1448-1396" | bc      #又減少了52字節
52
$ time ./hello
Hello World

real    0m0.130s
user    0m0.004s
sys     0m0.000s
$ strip -R .gnu.version_r hello   #刪除.gnu.version_r就不工作了
$ time ./hello
./hello: error while loading shared libraries: ./hello: unsupported version 0 of Verneed record
```

通過刪除各個節區可以查看哪些節區對程序來說是必須的，不過有些節區雖然並不影響程序的運行卻可能會影響程序的執行效率，這個可以上面的運行時間看出個大概。
通過刪除兩個“沒用”的節區，我們又減少了 `52+612`，即 664 字節。

<span id="toc_3928_6176_8"></span>
### 刪除可執行文件的節區表

用普通的工具沒有辦法刪除節區表，但是參考資料[\[1\]][1]的作者已經寫了這樣一個工具。你可以從[這裡](http://www.muppetlabs.com/~breadbox/software/elfkickers.html)下載到那個工具，它是該作者寫的一序列工具 `ELFkickers` 中的一個。

下載並編譯（**注**：1.0 之前的版本才支持 32 位和正常編譯，新版本在代碼中明確限定了數據結構為 `Elf64`）：

```
$ git clone https://github.com/BR903/ELFkickers
$ cd ELFkickers/sstrip/
$ git checkout f0622afa    # 檢出 1.0 版
$ make
```

然後複製到 `/usr/bin` 下，下面用它來刪除節區表。

```
$ sstrip hello      #刪除ELF可執行文件的節區表
$ ./hello           #還是可以正常運行，說明節區表對可執行文件的運行沒有任何影響
Hello World
$ wc -c hello       #大小隻剩下708個字節了
708 hello
$ echo "1396-708" | bc  #又減少了688個字節。
688
```

通過刪除節區表又把可執行文件減少了 688 字節。現在回頭看看相對於 `gcc` 默認產生的可執行文件，通過刪除一些節區和節區表到底減少了多少字節？減幅達到了多少？

```
$ echo "6442-708" | bc   #
5734
$ echo "(6442-708)/6442" | bc -l
.89009624340266997826
```

減少了 5734 多字節，減幅將近 `90%`，這說明：對於一個簡短的 `hello.c` 程序而言，`gcc` 引入了將近 `90%` 的對程序運行沒有影響的數據。雖然通過刪除節區和節區表，使得最終的文件只有 708 字節，但是打印一個 `Hello World` 真的需要這麼多字節麼？事實上未必，因為：

- 打印一段 `Hello World` 字符串，我們無須調用 `printf`，也就無須包含動態連接庫，因此 `.interp`，`.dynamic` 等節區又可以去掉。為什麼？我們可以直接使用系統調用 `(sys_write)來打印字符串。
- 另外，我們無須把 `Hello World` 字符串存放到可執行文件中？而是讓用戶把它當作參數輸入。

下面，繼續進行可執行文件的“減肥”。

<span id="toc_3928_6176_9"></span>
## 用匯編語言來重寫"Hello World"（76字節）

<span id="toc_3928_6176_10"></span>
### 採用默認編譯

先來看看 `gcc` 默認產生的彙編代碼情況。通過 `gcc` 的 `-S` 選項可得到彙編代碼。

```
$ cat hello.c  #這個是使用_exit和printf函數的版本
#include <stdio.h>      /* printf */
#include <unistd.h>     /* _exit */

int main()
{
	printf("Hello World\n");
	_exit(0);
}
$ gcc -S hello.c    #生成彙編
$ cat hello.s       #這裡是彙編代碼
	.file   "hello.c"
	.section        .rodata
.LC0:
	.string "Hello World"
	.text
.globl main
	.type   main, @function
main:
	leal    4(%esp), %ecx
	andl    $-16, %esp
	pushl   -4(%ecx)
	pushl   %ebp
	movl    %esp, %ebp
	pushl   %ecx
	subl    $4, %esp
	movl    $.LC0, (%esp)
	call    puts
	movl    $0, (%esp)
	call    _exit
	.size   main, .-main
	.ident  "GCC: (GNU) 4.1.3 20070929 (prerelease) (Ubuntu 4.1.2-16ubuntu2)"
	.section        .note.GNU-stack,"",@progbits
$ gcc -o hello hello.s   #看看默認產生的代碼大小
$ wc -c hello
6523 hello
```

<span id="toc_3928_6176_11"></span>
### 刪除掉彙編代碼中無關緊要內容

現在對彙編代碼 `hello.s` 進行簡單的處理得到，

```
.LC0:
	.string "Hello World"
	.text
.globl main
	.type   main, @function
main:
	leal    4(%esp), %ecx
	andl    $-16, %esp
	pushl   -4(%ecx)
	pushl   %ebp
	movl    %esp, %ebp
	pushl   %ecx
	subl    $4, %esp
	movl    $.LC0, (%esp)
	call    puts
	movl    $0, (%esp)
	call    _exit
```

再編譯看看，

```
$ gcc -o hello.o hello.s
$ wc -c hello
6443 hello
$ echo "6523-6443" | bc   #僅僅減少了80個字節
80
```

<span id="toc_3928_6176_12"></span>
### 不默認編譯並刪除掉無關節區和節區表

如果不採用默認編譯呢並且刪除掉對程序運行沒有影響的節區和節區表呢？

```
$ sed -i -e "s/main/_start/g" hello.s   #因為沒有初始化，所以得直接進入代碼，替換main為_start
$ as --32 -o  hello.o hello.s
$ ld -melf_i386 -o hello hello.o --dynamic-linker /lib/ld-linux.so.2 -L /usr/lib -lc
$ ./hello
hello world!
$ wc -c hello
1812 hello
$ echo "6443-1812" | bc -l   #和之前的實驗類似，也減少了4k左右
4631
$ readelf -l hello | grep "\ [0-9][0-9]\ "
   00
   01     .interp
   02     .interp .hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.plt .plt .text
   03     .dynamic .got.plt
   04     .dynamic
$ strip -R .hash hello
$ strip -R .gnu.version hello
$ wc -c hello
1200 hello
$ sstrip hello
$ wc -c hello  #這個結果比之前的708（在刪除所有垃圾信息以後）個字節少了708-676，即32個字節
676 hello
$ ./hello
Hello World
```

容易發現這 32 字節可能跟節區 `.rodata` 有關係，因為剛才在鏈接完以後查看節區信息時，並沒有 `.rodata` 節區。

<span id="toc_3928_6176_13"></span>
### 用系統調用取代庫函數

前面提到，實際上還可以不用動態連接庫中的 `printf` 函數，也不用直接調用 `_exit`，而是在彙編裡頭使用系統調用，這樣就可以去掉和動態連接庫關聯的內容。如果想了解如何在彙編中使用系統調用，請參考資料 [\[9\]][9]。使用系統調用重寫以後得到如下代碼，

```
.LC0:
	.string "Hello World\xa\x0"
	.text
.global _start
_start:
	xorl   %eax, %eax
	movb   $4, %al                  #eax = 4, sys_write(fd, addr, len)
	xorl   %ebx, %ebx
	incl   %ebx                     #ebx = 1, standard output
	movl   $.LC0, %ecx              #ecx = $.LC0, the address of string
	xorl   %edx, %edx
	movb   $13, %dl                 #edx = 13, the length of .string
	int    $0x80
	xorl   %eax, %eax
	movl   %eax, %ebx               #ebx = 0
	incl   %eax                     #eax = 1, sys_exit
	int    $0x80
```

現在編譯就不再需要動態鏈接器 `ld-linux.so` 了，也不再需要鏈接任何庫。

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o
$ readelf -l hello

Elf file type is EXEC (Executable file)
Entry point 0x8048062
There are 1 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x08048000 0x08048000 0x0007b 0x0007b R E 0x1000

 Section to Segment mapping:
  Segment Sections...
   00     .text
$ sstrip hello
$ ./hello           #完全可以正常工作
Hello World
$ wc -c hello
123 hello
$ echo "676-123" | bc   #相對於之前，已經只需要123個字節了，又減少了553個字節
553
```

可以看到效果很明顯，只剩下一個 `LOAD` 段，它對應 `.text` 節區。

<span id="toc_3928_6176_14"></span>
### 把字符串作為參數輸入

不過是否還有辦法呢？把 `Hello World` 作為參數輸入，而不是硬編碼在文件中。所以如果處理參數的代碼少於 `Hello World` 字符串的長度，那麼就可以達到減少目標文件大小的目的。

先來看一個能夠打印程序參數的彙編語言程序，它來自參考資料[\[9\]][9]。

```
.text
.globl _start

_start:
	popl    %ecx            # argc
vnext:
	popl    %ecx            # argv
	test    %ecx, %ecx      # 空指針表明結束
	jz      exit
	movl    %ecx, %ebx
	xorl    %edx, %edx
strlen:
	movb    (%ebx), %al
	inc     %edx
	inc     %ebx
	test    %al, %al
	jnz     strlen
	movb    $10, -1(%ebx)
	movl    $4, %eax        # 系統調用號(sys_write)
	movl    $1, %ebx        # 文件描述符(stdout)
	int     $0x80
	jmp     vnext
exit:
	movl    $1,%eax         # 系統調用號(sys_exit)
	xorl    %ebx, %ebx      # 退出代碼
	int     $0x80
	ret
```

編譯看看效果，

```
$ as --32 -o args.o args.s
$ ld -melf_i386 -o args args.o
$ ./args "Hello World"  #能夠打印輸入的字符串，不錯
./args
Hello World
$ sstrip args
$ wc -c args           #處理以後只剩下130字節
130 args
```

可以看到，這個程序可以接收用戶輸入的參數並打印出來，不過得到的可執行文件為 130 字節，比之前的 123 個字節還多了 7 個字節，看看還有改進麼？分析上面的代碼後，發現，原來的代碼有些地方可能進行優化，優化後得到如下代碼。

```
.global _start
_start:
	popl %ecx        #彈出argc
vnext:
	popl %ecx        #彈出argv[0]的地址
	test %ecx, %ecx  #空指針表明結束
	jz exit
	movl %ecx, %ebx  #複製字符串地址到ebx寄存器
	xorl %edx, %edx  #把字符串長度清零
strlen:                         #求輸入字符串的長度
	movb (%ebx), %al        #複製字符到al，以便判斷是否為字符串結束符\0
	inc %edx                #edx存放每個當前字符串的長度
	inc %ebx                #ebx存放每個當前字符的地址
	test %al, %al           #判斷字符串是否結束，即是否遇到\0
	jnz strlen
	movb $10, -1(%ebx)      #在字符串末尾插入一個換行符\0xa
	xorl %eax, %eax
	movb $4, %al            #eax = 4, sys_write(fd, addr, len)
	xorl %ebx, %ebx
	incl %ebx               #ebx = 1, standard output
	int $0x80
	jmp vnext
exit:
	xorl %eax, %eax
	movl %eax, %ebx                 #ebx = 0
	incl %eax               #eax = 1, sys_exit
	int $0x80
```

再測試（記得先重新彙編、鏈接並刪除沒用的節區和節區表）。

```
$ wc -c hello
124 hello
```

現在只有 124 個字節，不過還是比 123 個字節多一個，還有什麼優化的辦法麼？

先來看看目前 `hello` 的功能，感覺不太符合要求，因為只需要打印 `Hello World`，所以不必處理所有的參數，僅僅需要接收並打印一個參數就可以。這樣的話，把 `jmp vnext`（2 字節）這個循環去掉，然後在第一個 `pop %ecx` 語句之前加一個 `pop %ecx`（1 字節）語句就可以。

```
.global _start
_start:
	popl %ecx
	popl %ecx        #彈出argc[0]的地址
	popl %ecx        #彈出argv[1]的地址
	test %ecx, %ecx
	jz exit
	movl %ecx, %ebx
	xorl %edx, %edx
strlen:
	movb (%ebx), %al
	inc %edx
	inc %ebx
	test %al, %al
	jnz strlen
	movb $10, -1(%ebx)
	xorl %eax, %eax
	movb $4, %al
	xorl %ebx, %ebx
	incl %ebx
	int $0x80
exit:
	xorl %eax, %eax
	movl %eax, %ebx
	incl %eax
	int $0x80
```

現在剛好 123 字節，和原來那個代碼大小一樣，不過仔細分析，還是有減少代碼的餘地：因為在這個代碼中，用了一段額外的代碼計算字符串的長度，實際上如果僅僅需要打印 `Hello World`，那麼字符串的長度是固定的，即 12 。所以這段代碼可去掉，與此同時測試字符串是否為空也就沒有必要（不過可能影響代碼健壯性！），當然，為了能夠在打印字符串後就換行，在串的末尾需要加一個回車（`$10`）並且設置字符串的長度為 `12+1`，即 13，

```
.global _start
_start:
	popl %ecx
	popl %ecx
	popl %ecx
	movb $10,12(%ecx) #在Hello World的結尾加一個換行符
	xorl %edx, %edx
	movb $13, %dl
	xorl %eax, %eax
	movb $4, %al
	xorl %ebx, %ebx
	incl %ebx
	int $0x80
	xorl %eax, %eax
	movl %eax, %ebx
	incl %eax
	int $0x80
```

再看看效果，

```
$ wc -c hello
111 hello
```

<span id="toc_3928_6176_15"></span>
### 寄存器賦值重用

現在只剩下 111 字節，比剛才少了 12 字節。貌似到了極限？還有措施麼？

還有，仔細分析發現：系統調用 `sys_exit` 和 `sys_write` 都用到了 `eax` 和 `ebx` 寄存器，它們之間剛好有那麼一點巧合：

- sys_exit 調用時，`eax` 需要設置為 1，`ebx` 需要設置為 0 。
- sys_write 調用時，`ebx` 剛好是 1 。

因此，如果在 `sys_exit` 調用之前，先把 `ebx` 複製到 `eax` 中，再對 `ebx` 減一，則可減少兩個字節。

不過，因為標準輸入、標準輸出和標準錯誤都指向終端，如果往標準輸入寫入一些東西，它還是會輸出到標準輸出上，所以在上述代碼中如果在 `sys_write` 之前 `ebx` 設置為 0，那麼也可正常往屏幕上打印 `Hello World`，這樣的話，`sys_exit` 調用前就沒必要修改 `ebx`，而僅需把 `eax` 設置為 1，這樣就可減少 3 個字節。

```
.global _start
_start:
	popl %ecx
	popl %ecx
	popl %ecx
	movb $10,12(%ecx)
	xorl %edx, %edx
	movb $13, %dl
	xorl %eax, %eax
	movb $4, %al
	xorl %ebx, %ebx
	int $0x80
	xorl %eax, %eax
	incl %eax
	int $0x80
```

看看效果，

```
$ wc -c hello
108 hello
```

現在看一下純粹的指令還有多少？

```
$ readelf -h hello | grep Size
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Size of section headers:           0 (bytes)
$  echo "108-52-32" | bc
24
```

<span id="toc_3928_6176_16"></span>
### 通過文件名傳遞參數

對於標準的 `main` 函數的兩個參數，文件名實際上作為第二個參數（數組）的第一個元素傳入，如果僅僅是為了打印一個字符串，那麼可以打印文件名本身。例如，要打印 `Hello World`，可以把文件名命名為 `Hello World` 即可。

這樣地話，代碼中就可以刪除掉一條 `popl` 指令，減少 1 個字節，變成 107 個字節。

```
.global _start
_start:
	popl %ecx
	popl %ecx
	movb $10,12(%ecx)
	xorl %edx, %edx
	movb $13, %dl
	xorl %eax, %eax
	movb $4, %al
	xorl %ebx, %ebx
	int $0x80
	xorl %eax, %eax
	incl %eax
	int $0x80
```

看看效果，

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o
$ sstrip hello
$ wc -c hello
107
$ mv hello "Hello World"
$ export PATH=./:$PATH
$ Hello\ World
Hello World
```

<span id="toc_3928_6176_17"></span>
### 刪除非必要指令

在測試中發現，`edx`，`eax`，`ebx` 的高位即使不初始化，也常為 0，如果不考慮健壯性（僅這裡實驗用，實際使用中必須考慮健壯性），幾條 `xorl` 指令可以移除掉。

另外，如果只是為了演示打印字符串，完全可以不用打印換行符，這樣下來，代碼可以綜合優化成如下幾條指令：

```
.global _start
_start:
	popl %ecx	# argc
	popl %ecx	# argv[0]
	movb $5, %dl	# 設置字符串長度
	movb $4, %al	# eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
	int $0x80
	movb $1, %al
	int $0x80
```

看看效果：

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o
$ sstrip hello
$ wc -c hello
96
```

<span id="toc_3928_6176_18"></span>
## 合併代碼段、程序頭和文件頭（52字節）

<span id="toc_3928_6176_19"></span>
### 把代碼段移入文件頭

純粹的指令只有 `96-84=12` 個字節了，還有辦法再減少目標文件的大小麼？如果看了參考資料 [\[1\]][1]，看樣子你又要蠢蠢欲動了：這 12 個字節是否可以插入到文件頭部或程序頭部？如果可以那是否意味著還可減少可執行文件的大小呢？現在來比較一下這三部分的十六進制內容。

```
$ hexdump -C hello -n 52     #文件頭(52bytes)
00000000  7f 45 4c 46 01 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  02 00 03 00 01 00 00 00  54 80 04 08 34 00 00 00  |........T...4...|
00000020  00 00 00 00 00 00 00 00  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00                                       |....|
00000034
$ hexdump -C hello -s 52 -n 32    #程序頭(32bytes)
00000034  01 00 00 00 00 00 00 00  00 80 04 08 00 80 04 08  |................|
00000044  6c 00 00 00 6c 00 00 00  05 00 00 00 00 10 00 00  |l...l...........|
00000054
$ hexdump -C hello -s 84          #實際代碼部分(12bytes)
00000054  59 59 b2 05 b0 04 cd 80  b0 01 cd 80              |YY..........|
00000060
```

從上面結果發現 `ELF` 文件頭部和程序頭部還有好些空洞（0），是否可以把指令字節分散放入到那些空洞裡或者是直接覆蓋掉那些系統並不關心的內容？抑或是把代碼壓縮以後放入可執行文件中，並在其中實現一個解壓縮算法？還可以是通過一些代碼覆蓋率測試工具（`gcov`，`prof`）對你的代碼進行優化？

在繼續介紹之前，先來看一個 `dd` 工具，可以用來直接“編輯” `ELF` 文件，例如，

直接往指定位置寫入 `0xff` ：

```
$ hexdump -C hello -n 16	# 寫入前，elf文件前16個字節
00000000  7f 45 4c 46 01 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010
$ echo -ne "\xff" | dd of=hello bs=1 count=1 seek=15 conv=notrunc	# 把最後一個字節0覆蓋掉
1+0 records in
1+0 records out
1 byte (1 B) copied, 3.7349e-05 s, 26.8 kB/s
$ hexdump -C hello -n 16	# 寫入後果然被覆蓋
00000000  7f 45 4c 46 01 01 01 00  00 00 00 00 00 00 00 ff  |.ELF............|
00000010
```

- `seek=15` 表示指定寫入位置為第 15 個（從第 0 個開始）
- `conv=notrunc` 選項表示要保留寫入位置之後的內容，默認情況下會截斷。
- `bs=1` 表示一次讀/寫 1 個
- `count=1` 表示總共寫 1 次

覆蓋多個連續的值：

把第 12，13，14，15 連續 4 個字節全部賦值為 `0xff` 。

```
$ echo -ne "\xff\xff\xff\xff" | dd of=hello bs=1 count=4 seek=12 conv=notrunc
$ hexdump -C hello -n 16
00000000  7f 45 4c 46 01 01 01 00  00 00 00 00 ff ff ff ff  |.ELF............|
00000010
```

下面，通過往文件頭指定位置寫入 `0xff` 確認哪些部分對於可執行文件的執行是否有影響？這裡是逐步測試後發現依然能夠執行的情況：

```
$ hexdump -C hello
00000000  7f 45 4c 46 ff ff ff ff  ff ff ff ff ff ff ff ff  |.ELF............|
00000010  02 00 03 00 ff ff ff ff  54 80 04 08 34 00 00 00  |........T...4...|
00000020  ff ff ff ff ff ff ff ff  34 00 20 00 01 00 ff ff  |........4. .....|
00000030  ff ff ff ff 01 00 00 00  00 00 00 00 00 80 04 08  |................|
00000040  00 80 04 08 60 00 00 00  60 00 00 00 05 00 00 00  |....`...`.......|
00000050  00 10 00 00 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |....YY..........|
00000060
```

可以發現，文件頭部分，有 30 個字節即使被篡改後，該可執行文件依然可以正常執行。這意味著，這 30 字節是可以寫入其他代碼指令字節的。而我們的實際代碼指令只剩下 12 個，完全可以直接移到前 12 個 `0xff` 的位置，即從第 4 個到第 15 個。

而代碼部分的起始位置，通過 `readelf -h` 命令可以看到：

```
$ readelf -h hello | grep "Entry"
  Entry point address:               0x8048054
```

上面地址的最後兩位 `0x54=84` 就是代碼在文件中的偏移，也就是剛好從程序頭之後開始的，也就是用文件頭（52）+程序頭（32）個字節開始的 12 字節覆蓋到第 4 個字節開始的 12 字節內容即可。

上面的 `dd` 命令從 `echo` 命令獲得輸入，下面需要通過可執行文件本身獲得輸入，先把代碼部分移過去：

```
$ dd if=hello of=hello bs=1 skip=84 count=12 seek=4 conv=notrunc
12+0 records in
12+0 records out
12 bytes (12 B) copied, 4.9552e-05 s, 242 kB/s
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 01 00 00 00  54 80 04 08 34 00 00 00  |........T...4...|
00000020  00 00 00 00 00 00 00 00  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00 01 00 00 00  00 00 00 00 00 80 04 08  |................|
00000040  00 80 04 08 60 00 00 00  60 00 00 00 05 00 00 00  |....`...`.......|
00000050  00 10 00 00 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |....YY..........|
00000060
```

接著把代碼部分截掉：

```
$ dd if=hello of=hello bs=1 count=1 skip=84 seek=84
0+0 records in
0+0 records out
0 bytes (0 B) copied, 1.702e-05 s, 0.0 kB/s
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 01 00 00 00  54 80 04 08 34 00 00 00  |........T...4...|
00000020  00 00 00 00 00 00 00 00  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00 01 00 00 00  00 00 00 00 00 80 04 08  |................|
00000040  00 80 04 08 60 00 00 00  60 00 00 00 05 00 00 00  |....`...`.......|
00000050  00 10 00 00                                       |....|
00000054
```

這個時候還不能執行，因為代碼在文件中的位置被移動了，相應地，文件頭中的 `Entry point address`，即文件入口地址也需要被修改為 `0x8048004` 。

即需要把 `0x54` 所在的第 24 個字節修改為 `0x04` ：

```
$ echo -ne "\x04" | dd of=hello bs=1 count=1 seek=24 conv=notrunc
1+0 records in
1+0 records out
1 byte (1 B) copied, 3.7044e-05 s, 27.0 kB/s
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 01 00 00 00  04 80 04 08 34 00 00 00  |............4...|
00000020  84 00 00 00 00 00 00 00  34 00 20 00 01 00 28 00  |........4. ...(.|
00000030  05 00 02 00 01 00 00 00  00 00 00 00 00 80 04 08  |................|
00000040  00 80 04 08 60 00 00 00  60 00 00 00 05 00 00 00  |....`...`.......|
00000050  00 10 00 00
```

修改後就可以執行了。

<span id="toc_3928_6176_20"></span>
### 把程序頭移入文件頭

程序頭部分經過測試發現基本上都不能修改並且需要是連續的，程序頭有 32 個字節，而文件頭中連續的 `0xff` 可以被篡改的只有從第 46 個開始的 6 個了，另外，程序頭剛好是 `01 00` 開頭，而第 44，45 個剛好為 `01 00`，這樣地話，這兩個字節文件頭可以跟程序頭共享，這樣地話，程序頭就可以往文件頭裡頭移動 8 個字節了。

```
$ dd if=hello of=hello bs=1 skip=52 seek=44 count=32 conv=notrunc
```

再把最後 8 個沒用的字節刪除掉，保留 `84-8=76` 個字節：

```
$ dd if=hello of=hello bs=1 skip=76 seek=76
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 01 00 00 00  04 80 04 08 34 00 00 00  |............4...|
00000020  84 00 00 00 00 00 00 00  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00 00 80 04 08  00 80 04 08 60 00 00 00  |............`...|
00000040  60 00 00 00 05 00 00 00  00 10 00 00              |`...........|
0000004c
```

另外，還需要把文件頭中程序頭的位置信息改為 44，即第 28 個字節，原來是 `0x34`，即 52 的位置。

```
$ echo "obase=16;ibase=10;44" | bc	# 先把44轉換是16進制的0x2C
2C
$ echo -ne "\x2C" | dd of=hello bs=1 count=1 seek=28 conv=notrunc	# 修改文件頭
1+0 records in
1+0 records out
1 byte (1 B) copied, 3.871e-05 s, 25.8 kB/s
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 01 00 00 00  04 80 04 08 2c 00 00 00  |............,...|
00000020  84 00 00 00 00 00 00 00  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00 00 80 04 08  00 80 04 08 60 00 00 00  |............`...|
00000040  60 00 00 00 05 00 00 00  00 10 00 00              |`...........|
0000004c
```

修改後即可執行了，目前只剩下 76 個字節：

```
$ wc -c hello
76
```

<span id="toc_3928_6176_21"></span>
### 在非連續的空間插入代碼

另外，還有 12 個字節可以放代碼，見 `0xff` 的地方：

```
$ hexdump -C hello
00000000  7f 45 4c 46 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |.ELFYY..........|
00000010  02 00 03 00 ff ff ff ff  04 80 04 08 2c 00 00 00  |............,...|
00000020  ff ff ff ff ff ff ff ff  34 00 20 00 01 00 00 00  |........4. .....|
00000030  00 00 00 00 00 80 04 08  00 80 04 08 60 00 00 00  |............`...|
00000040  60 00 00 00 05 00 00 00  00 10 00 00              |`...........|
0000004c
```

不過因為空間不是連續的，需要用到跳轉指令作為跳板利用不同的空間。

例如，如果要利用後面的 `0xff` 的空間，可以把第 14，15 位置的 `cd 80` 指令替換為一條跳轉指令，比如跳轉到第 20 個字節的位置，從跳轉指令之後的 16 到 20 剛好 4 個字節。

然後可以參考 [X86 指令編碼表][15]（也可以寫成彙編生成可執行文件後用 `hexdump` 查看），可以把 `jmp` 指令編碼為： `0xeb 0x04` 。

```
$ echo -ne "\xeb\x04" | dd of=hello bs=1 count=2 seek=14 conv=notrunc
```

然後把原來位置的 `cd 80` 移動到第 20 個字節開始的位置：

```
$ echo -ne "\xcd\x80" | dd of=hello bs=1 count=2 seek=20 conv=notrunc
```

依然可以執行，類似地可以利用更多非連續的空間。

<span id="toc_3928_6176_22"></span>
### 把程序頭完全合入文件頭

在閱讀參考資料 [\[1\]][1]後，發現有更多深層次的探討，通過分析 Linux 系統對 `ELF` 文件頭部和程序頭部的解析，可以更進一步合併程序頭和文件頭。

該資料能夠把最簡的 `ELF` 文件（簡單返回一個數值）壓縮到 45 個字節，真地是非常極端的努力，思路可以充分借鑑。在充分理解原文的基礎上，我們進行更細緻地梳理。

首先對 `ELF` 文件頭部和程序頭部做更徹底的理解，並具體到每一個字節的含義以及在 Linux 系統下的實際解析情況。

先來看看 `readelf -a` 的結果：

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o
$ sstrip hello
$ readelf -a hello
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x8048054
  Start of program headers:          52 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         1
  Size of section headers:           0 (bytes)
  Number of section headers:         0
  Section header string table index: 0

There are no sections in this file.

There are no sections to group in this file.

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000000 0x08048000 0x08048000 0x00060 0x00060 R E 0x1000
```

然後結合 `/usr/include/linux/elf.h` 分別做詳細註解。

首先是 52 字節的 `Elf` 文件頭的結構體 `elf32_hdr`：


  變量類型     |  變量名           | 字節  | 說明                   | 類型
 --------------|-------------------|-------|------------------------|-------
  unsigned char|e_ident[EI_NIDENT] |16     | .ELF 前四個標識文件類型| 必須
  Elf32_Half   |e_type             |2      | 指定為可執行文件 | 必須
  Elf32_Half   |e_machine          |2      | 指示目標機類型，例如：Intel 386 | 必須
  Elf32_Word   |e_version          |4      | 當前只有一個版本存在，被忽略了 | ~~可篡改~~
  Elf32_Addr   |e_entry            |4      | 代碼入口=加載地址(p_vaddr+.text偏移) | **可調整**
  Elf32_Off    |e_phoff            |4      | 程序頭 Phdr 的偏移地址，用於加載代碼| 必須
  Elf32_Off    |e_shoff            |4      | 所有節區相關信息對文件執行無效 | ~~可篡改~~
  Elf32_Word   |e_flags            |4      | Intel 架構未使用 | ~~可篡改~~
  Elf32_Half   |e_ehsize           |2      | 文件頭大小，Linux 沒做校驗 | ~~可篡改~~
  Elf32_Half   |e_phentsize        |2      | 程序頭入口大小，新內核有用 | 必須
  Elf32_Half   |e_phnum            |2      | 程序頭入口個數 | 必須
  Elf32_Half   |e_shentsize        |2      | 所有節區相關信息對文件執行無效 | ~~可篡改~~
  Elf32_Half   |e_shnum            |2      | 所有節區相關信息對文件執行無效 | ~~可篡改~~
  Elf32_Half   |e_shstrndx         |2      | 所有節區相關信息對文件執行無效 | ~~可篡改~~

其次是 32 字節的程序頭（Phdr）的結構體 `elf32_phdr`：

  變量類型     |  變量名  | 字節  | 說明           | 類型
 --------------|----------|-------|----------------|-------
  Elf32_Word   |p_type    |4      | 標記為可加載段 | 必須
  Elf32_Off    |p_offset  |4      | 相對程序頭的偏移地址| 必須
  Elf32_Addr   |p_vaddr   |4      | 加載地址, 0x0~0x80000000，頁對齊 | **可調整**
  Elf32_Addr   |p_paddr   |4      | 物理地址，暫時沒用 | ~~可篡改~~
  Elf32_Word   |p_filesz  |4      | 加載的文件大小，>=real size | **可調整**
  Elf32_Word   |p_memsz   |4      | 加載所需內存大小，>= p_filesz | **可調整**
  Elf32_Word   |p_flags   |4      | 權限:read(4),exec(1), 其中一個暗指另外一個|**可調整**
  Elf32_Word   |p_align   |4      | PIC(共享庫需要)，對執行文件無效 | ~~可篡改~~

接著，咱們把 Elf 中的文件頭和程序頭部分**可調整**和~~可篡改~~的字節（52 + 32 = 84個）全部用特別的字體標記出來。

```
$ hexdump -C hello -n 84
```


> 00000000  7f 45 4c 46 ~~01 01 01 00  00 00 00 00 00 00 00 00~~

> 00000010  02 00 03 00 ~~01 00 00 00~~ **54 80 04 08 34** 00 00 00

> 00000020  ~~84 00 00 00 00 00 00 00~~  34 00 20 00 01 00 ~~28 00~~

> 00000030  ~~05 00 02 00~~|01 00 00 00  00 00 00 00 **00 80 04 08**

> 00000040  ~~00 80 04 08~~ **60 00 00 00  60 00 00 00 05 00 00 00**

> 00000050  ~~00 10 00 00~~

> 00000054

上述 `|` 線之前為文件頭，之後為程序頭，之前的 `000000xx` 為偏移地址。

如果要把程序頭徹底合併進文件頭。從上述信息綜合來看，文件頭有 4 處必須保留，結合資料 [\[1\]][1]，經過對比發現，如果把第 4 行開始的程序頭往上平移 3 行，也就是：


> 00000000  ========= ~~01 01 01 00  00 00 00 00 00 00 00 00~~

> 00000010  02 00 03 00 ~~01 00 00 00~~ **54 80 04 08 34** 00 00 00

> 00000020  ~~84 00 00 00~~

> 00000030  ========= 01 00 00 00  00 00 00 00 **00 80 04 08**

> 00000040  ~~00 80 04 08~~ **60 00 00 00  60 00 00 00 05 00 00 00**

> 00000050  ~~00 10 00 00~~

> 00000054

把可直接合並的先合併進去，效果如下：

（文件頭）

> 00000000  ========= 01 00 00 00  00 00 00 00 **00 80 04 08** (^^ p_vaddr)

> 00000010  02 00 03 00 ~~60 00 00 00~~ **54 80 04 08 34** 00 00 00

> 00000020  =================== ^^ e_entry ^^ e_phoff

（程序頭）

> 00000030  ========= 01 00 00 00  00 00 00 00 **00 80 04 08** (^^ p_vaddr)

> 00000040  02 00 03 00 **60 00 00 00  60 00 00 00 05 00 00 00**

> 00000050  =========  ^^ p_filesz   ^^ p_memsz  ^^p_flags

> 00000054

接著需要設法處理好可調整的 6 處，可以逐個解決，從易到難。

* 首先，合併 `e_phoff` 與 `p_flags`

在合併程序頭以後，程序頭的偏移地址需要修改為 4，即文件的第 4 個字節開始，也就是說 `e_phoff` 需要修改為 04。

而恰好，`p_flags` 的 `read(4)` 和 `exec(1)` 可以只選其一，所以，只保留 `read(4)` 即可，剛好也為 04。

合併後效果如下：

（文件頭）

> 00000000  ========= 01 00 00 00  00 00 00 00 **00 80 04 08** (^^ p_vaddr)

> 00000010  02 00 03 00 ~~60 00 00 00~~ **54 80 04 08** 04 00 00 00

> 00000020  =================== ^^ e_entry

（程序頭）

> 00000030  ========= 01 00 00 00  00 00 00 00 **00 80 04 08** (^^ p_vaddr)

> 00000040  02 00 03 00 **60 00 00 00  60 00 00 00** 04 00 00 00

> 00000050  =========  ^^ p_filesz   ^^ p_memsz

> 00000054

* 接下來，合併 `e_entry`，`p_filesz`, `p_memsz`  和 `p_vaddr`

從早前的分析情況來看，這 4 個變量基本都依賴 `p_vaddr`，也就是程序的加載地址，大體的依賴關係如下：

```
e_entry = p_vaddr + text offset = p_vaddr + 84 = p_vaddr + 0x54

p_memsz = e_entry

p_memsz >= p_filesz，可以簡單取 p_filesz = p_memsz

p_vaddr = page alignment
```

所以，首先需要確定 `p_vaddr`，通過測試，發現`p_vaddr` 最低必須有 64k，也就是 0x00010000，對應到 `hexdump` 的 `little endian` 導出結果，則為 `00 00 01 00`。

需要注意的是，為了儘量少了分配內存，我們選擇了一個最小的`p_vaddr`，如果申請的內存太大，系統將無法分配。

接著，計算出另外 3 個變量：

```
e_entry = 0x00010000 + 0x54 = 0x00010054 即 54 00 01 00
p_memsz = 54 00 01 00
p_filesz = 54 00 01 00
```

完全合併後，修改如下：

（文件頭）

> 00000000  =========   01 00 00 00  00 00 00 00  00 00 01 00

> 00000010  02 00 03 00 54 00 01 00  54 00 01 00  04 00 00 00

> 00000020  ========

好了，直接把內容燒入：

```
$ echo -ne "\x01\x00\x00\x00\x00\x00\x00\x00" \
	   "\x00\x00\x01\x00\x02\x00\x03\x00" \
	   "\x54\x00\x01\x00\x54\x00\x01\x00\x04" |\
	   tr -d ' ' |\
    dd of=hello bs=1 count=25 seek=4 conv=notrunc
```

截掉代碼（52 + 32 + 12 = 96）之後的所有內容，查看效果如下：

```
$ dd if=hello of=hello bs=1 count=1 skip=96 seek=96
$ hexdump -C hello -n 96
00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00  |.ELF............|
00000010  02 00 03 00 54 00 01 00  54 00 01 00 04 00 00 00  |....T...T.......|
00000020  84 00 00 00 00 00 00 00  34 00 20 00 01 00 28 00  |........4. ...(.|
00000030  05 00 02 00 01 00 00 00  00 00 00 00 00 80 04 08  |................|
00000040  00 80 04 08 60 00 00 00  60 00 00 00 05 00 00 00  |....`...`.......|
00000050  00 10 00 00 59 59 b2 05  b0 04 cd 80 b0 01 cd 80  |....YY..........|
00000060
```

最後的工作是查看文件頭中剩下的~~可篡改~~的內容，並把**代碼部分**合併進去，程序頭已經合入，不再顯示。

> 00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00

> 00000010  02 00 03 00 54 00 01 00  54 00 01 00 04 00 00 00

> 00000020  ~~84 00 00 00 00 00 00 00~~  34 00 20 00 01 00 ~~28 00~~

> 00000030  ~~05 00 02 00~~

> 00000040

> 00000050  ============= **59 59 b2 05  b0 04 cd 80 b0 01 cd 80**

> 00000060

我們的指令有 12 字節，~~可篡改~~的部分有 14 個字節，理論上一定放得下，不過因為把程序頭搬進去以後，這 14 個字節並不是連續，剛好可以用上我們之前的跳轉指令處理辦法來解決。

並且，加入 2 個字節的跳轉指令，剛好是 14 個字節，恰好把代碼也完全包含進了文件頭。

在預留好**跳轉指令**位置的前提下，我們把代碼部分先合併進去：

> 00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00

> 00000010  02 00 03 00 54 00 01 00  54 00 01 00 04 00 00 00

> 00000020  **59 59 b2 05  b0 04** ~~00 00~~  34 00 20 00 01 00 **cd 80**

> 00000030  **b0 01 cd 80**


接下來設計跳轉指令，跳轉指令需要從所在位置跳到第一個 **cd 80** 所在的位置，相距 6 個字節，根據 `jmp` 短跳轉的編碼規範，可以設計為 `0xeb 0x06`，填完後效果如下：


> 00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00

> 00000010  02 00 03 00 54 00 01 00  54 00 01 00 04 00 00 00

> 00000020  **59 59 b2 05  b0 04 eb 06** 34 00 20 00 01 00 **cd 80**

> 00000030  **b0 01 cd 80**

用 `dd` 命令寫入，分兩段寫入：

```
$ echo -ne "\x59\x59\xb2\x05\xb0\x04\xeb\x06" | \
    dd of=hello bs=1 count=8 seek=32 conv=notrunc

$ echo -ne "\xcd\x80\xb0\x01\xcd\x80" | \
    dd of=hello bs=1 count=6 seek=46 conv=notrunc
```

代碼合入以後，需要修改文件頭中的代碼的偏移地址，即 `e_entry`，也就是要把原來的偏移 84 (0x54) 修改為現在的偏移，即 0x20。

```
$ echo -ne "\x20" | dd of=hello bs=1 count=1 seek=24 conv=notrunc
```

修改完以後恰好把合併進的程序頭 `p_memsz`，也就是分配給文件的內存改小了，`p_filesz`也得相應改小。

```
$ echo -ne "\x20" | dd of=hello bs=1 count=1 seek=20 conv=notrunc
```

程序頭和代碼都已經合入，最後，把 52 字節之後的內容全部刪掉：

```
$ dd if=hello of=hello bs=1 count=1 skip=52 seek=52
$ hexdump -C hello
00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00  |.ELF............|
00000010  02 00 03 00 20 00 01 00  20 00 01 00 04 00 00 00  |....T...T.......|
00000020  59 59 b2 05 b0 04 eb 06  34 00 20 00 01 00 cd 80  |YY......4. .....|
00000030  b0 01 cd 80
$ export PATH=./:$PATH
$ hello
hello
```

**代碼**和~~程序頭~~部分合並進文件頭的彙總情況：

> 00000000  7f 45 4c 46 ~~01 00 00 00  00 00 00 00 00 00 01 00~~

> 00000010  ~~02 00 03 00 20 00 01 00  20 00 01 00 04 00 00 00~~

> 00000020  **~~59 59 b2 05~~ b0 04 eb 06**  34 00 20 00 01 00 **cd 80**

> 00000030  **b0 01 cd 80**


最後，我們的成績是：

```
$ wc -c hello
52
```

史上最小的可打印 `Hello World`（注：要完全打印得把代碼中的5該為13，並且把文件名該為該字符串） 的 `Elf` 文件是 52 個字節。打破了資料 [\[1\]][1] 作者創造的紀錄：

```
$ cd ELFkickers/tiny/
$ wc -c hello
59 hello
```
需要特別提到的是，該作者創造的最小可執行 Elf 是 45 個字節。

但是由於那個程序只能返回一個數值，代碼更簡短，剛好可以直接嵌入到文件頭中間，而文件末尾的 7 個 `0` 字節由於 Linux 加載時會自動填充，所以可以刪掉，所以最終的文件大小是 52 - 7 即 45 個字節。

其大體可實現如下：

```
.global _start
_start:
	mov $42, %bl   # 設置返回值為 42
	xor %eax, %eax # eax = 0
	inc %eax       # eax = eax+1, 設置系統調用號, sys_exit()
	int $0x80
```

保存為 ret.s，編譯和執行效果如下：

```
$ as --32 -o ret.o ret.s
$ ld -melf_i386 -o ret ret.o
$ ./ret
42
```

代碼字節數可這麼查看：

```
$ ld -melf_i386 --oformat=binary -o ret.bin ret.o
$ hexdump -C ret.bin
0000000  b3 2a 31 c0 40 cd 80
0000007
```

這裡只有 7 條指令，剛好可以嵌入，而最後的 6 個字節因為~~可篡改~~為 0，並且內核可自動填充 0，所以乾脆可以連續刪掉最後 7 個字節的 0：

> 00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00

> 00000010  02 00 03 00 54 00 01 00  54 00 01 00 04 00 00 00

> 00000020  **b3 2a 31 c0 40 cd 80** 00 34 00 20 00 01 00 00 00

> 00000030  00 00 00 00

可以直接用已經合併好程序頭的 `hello` 來做實驗，這裡一併截掉最後的 7 個 0 字節：

```
$ cp hello ret
$ echo -ne "\xb3\x2a\x31\xc0\x40\xcd\x80" |\
    dd of=ret bs=1 count=8 seek=32 conv=notrunc
$ dd if=ret of=hello bs=1 count=1 skip=45 seek=45
$ hexdump -C hello
00000000  7f 45 4c 46 01 00 00 00  00 00 00 00 00 00 01 00  |.ELF............|
00000010  02 00 03 00 20 00 01 00  20 00 01 00 04 00 00 00  |.... ... .......|
00000020  b3 2a 31 c0 40 cd 80 06  34 00 20 00 01           |.*1.@...4. ..|
0000002d
$ wc -c ret
45 ret
$ ./ret
$ echo $?
42
```

如果想快速構建該 `Elf` 文件，可以直接使用下述 Shell 代碼：

```
#!/bin/bash
#
# generate_ret_elf.sh -- Generate a 45 bytes Elf file
#
# $ bash generate_ret_elf.sh
# $ chmod a+x ret.elf
# $ ./ret.elf
# $ echo $?
# 42
#

ret="\x7f\x45\x4c\x46\x01\x00\x00\x00"
ret=${ret}"\x00\x00\x00\x00\x00\x00\x01\x00"
ret=${ret}"\x02\x00\x03\x00\x20\x00\x01\x00"
ret=${ret}"\x20\x00\x01\x00\x04\x00\x00\x00"
ret=${ret}"\xb3\x2a\x31\xc0\x40\xcd\x80\x06"
ret=${ret}"\x34\x00\x20\x00\x01"

echo -ne $ret > ret.elf
```

又或者是直接參照資料 [\[1\]][1] 的 `tiny.asm` 就行了，其代碼如下：

```
; ret.asm

  BITS 32

	        org     0x00010000

	        db      0x7F, "ELF"             ; e_ident
	        dd      1                                       ; p_type
	        dd      0                                       ; p_offset
	        dd      $$                                      ; p_vaddr
	        dw      2                       ; e_type        ; p_paddr
	        dw      3                       ; e_machine
	        dd      _start                  ; e_version     ; p_filesz
	        dd      _start                  ; e_entry       ; p_memsz
	        dd      4                       ; e_phoff       ; p_flags
  _start:
	        mov     bl, 42                  ; e_shoff       ; p_align
	        xor     eax, eax
	        inc     eax                     ; e_flags
	        int     0x80
	        db      0
	        dw      0x34                    ; e_ehsize
	        dw      0x20                    ; e_phentsize
	        db      1                       ; e_phnum
	                                        ; e_shentsize
	                                        ; e_shnum
	                                        ; e_shstrndx

  filesize      equ     $ - $$
```

編譯和運行效果如下：

```
$ nasm -f bin -o ret ret.asm
$ chmod +x ret
$ ./ret ; echo $?
42
$ wc -c ret
45 ret
```

下面也給一下本文精簡後的 `hello` 的 `nasm` 版本：

```
; hello.asm

  BITS 32

	        org     0x00010000

	        db      0x7F, "ELF"             ; e_ident
	        dd      1                                       ; p_type
	        dd      0                                       ; p_offset
	        dd      $$                                      ; p_vaddr
	        dw      2                       ; e_type        ; p_paddr
	        dw      3                       ; e_machine
	        dd      _start                  ; e_version     ; p_filesz
	        dd      _start                  ; e_entry       ; p_memsz
	        dd      4                       ; e_phoff       ; p_flags
  _start:
	        pop     ecx     ; argc          ; e_shoff       ; p_align
	        pop     ecx     ; argv[0]
	        mov     dl, 5   ; str len       ; e_flags
	        mov     al, 4   ; sys_write(fd, addr, len) : ebx, ecx, edx
	        jmp     _next   ; jump to next part of the code
	        dw      0x34                      ; e_ehsize
	        dw      0x20                      ; e_phentsize
	        dw      1                         ; e_phnum
  _next:        int     0x80    ; syscall         ; e_shentsize
	        mov     al, 1   ; eax=1,sys_exit  ; e_shnum
	        int     0x80    ; syscall         ; e_shstrndx

  filesize      equ     $ - $$
```

編譯和用法如下：

```
$ ./nasm -f bin -o hello hello.asm a.
$ chmod a+x hello
$ export PATH=./:$PATH
$ hello
hello
$ wc -c hello
52
```

經過一番努力，`AT&T` 的完整 binary 版本如下：

```
# hello.s
#
# as --32 -o hello.o hello.s
# ld -melf_i386 --oformat=binary -o hello hello.o
#

	.file "hello.s"
	.global _start, _load
	.equ   LOAD_ADDR, 0x00010000   # Page aligned load addr, here 64k
	.equ   E_ENTRY, LOAD_ADDR + (_start - _load)
	.equ   P_MEM_SZ, E_ENTRY
	.equ   P_FILE_SZ, P_MEM_SZ

_load:
	.byte  0x7F
	.ascii "ELF"                  # e_ident, Magic Number
	.long  1                                      # p_type, loadable seg
	.long  0                                      # p_offset
	.long  LOAD_ADDR                              # p_vaddr
	.word  2                      # e_type, exec  # p_paddr
	.word  3                      # e_machine, Intel 386 target
	.long  P_FILE_SZ              # e_version     # p_filesz
	.long  E_ENTRY                # e_entry       # p_memsz
	.long  4                      # e_phoff       # p_flags, read(exec)
	.text
_start:
	popl   %ecx    # argc         # e_shoff       # p_align
	popl   %ecx    # argv[0]
	mov    $5, %dl # str len      # e_flags
	mov    $4, %al # sys_write(fd, addr, len) : ebx, ecx, edx
	jmp    next    # jump to next part of the code
	.word  0x34                   # e_ehsize = 52
	.word  0x20                   # e_phentsize = 32
	.word  1                      # e_phnum = 1
	.text
_next:  int    $0x80   # syscall        # e_shentsize
	mov    $1, %al # eax=1,sys_exit # e_shnum
	int    $0x80   # syscall        # e_shstrndx
```

編譯和運行效果如下：

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 --oformat=binary -o hello hello.o
$ export PATH=./:$PATH
$ hello
hello
$ wc -c hello
52 hello
```

**注**：編譯時務必要加 `--oformat=binary` 參數，以便直接跟源文件構建一個二進制的 `Elf` 文件，否則會被 `ld` 默認編譯，自動填充其他內容。

<span id="toc_3928_6176_23"></span>
## 彙編語言極限精簡之道（45字節）

經過上述努力，我們已經完全把程序頭和代碼都融入了 52 字節的 `Elf` 文件頭，還可以再進一步嗎？

基於資料一，如果再要努力，只能設法把 `Elf` 末尾的 7 個 0 字節刪除，但是由於代碼已經把 `Elf` 末尾的 7 字節 0 字符都填滿了，所以要想在這一塊努力，只能繼續壓縮代碼。

繼續研究下代碼先：

```
.global _start
_start:
	popl %ecx	# argc
	popl %ecx	# argv[0]
	movb $5, %dl	# 設置字符串長度
	movb $4, %al	# eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
	int $0x80
	movb $1, %al
	int $0x80
```

查看對應的編碼：

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o --oformat=binary
$ hexdump -C hello
00000000  59 59 b2 05 b0 04 cd 80  b0 01 cd 80              |YY..........|
0000000c
```

每條指令對應的編碼映射如下：

  指令         |  編碼     | 說明
  -------------|-----------|-----------
  popl %ecx    |  59  	   | argc
  popl %ecx    |  59	   | argv[0]
  movb $5, %dl |  b2 05	   | 設置字符串長度
  movb $4, %al |  b0 04	   | eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
  int $0x80    |  cd 80    | 觸發系統調用
  movb $1, %al |  b0 01    | eax = 1, sys_exit
  int $0x80    |  cd 80    | 觸發系統調用

可以觀察到：

* `popl` 的指令編碼最簡潔。
* `int $0x80` 重複了兩次，而且每條都佔用了 2 字節
* `movb` 每條都佔用了 2 字節
* `eax` 有兩次賦值，每次佔用了 2 字節
* `popl %ecx` 取出的 argc 並未使用

根據之前通過參數傳遞字符串的想法，咱們是否可以考慮通過參數來設置變量呢？

理論上，傳入多個參數，通過 `pop` 彈出來賦予 `eax`, `ecx` 即可，但是實際上，由於從參數棧裡頭 `pop` 出來的參數是參數的地址，並不是參數本身，所以該方法行不通。

不過由於第一個參數取出的是數字，並且是參數個數，而且目前的那條 `popl %ecx` 取出的 `argc` 並沒有使用，那麼剛好可以用來設置 `eax`，替換後如下：

```
.global _start
_start:
	popl %eax    # eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
	popl %ecx    # argv[0], 字符串
	movb $5, %dl # 設置字符串長度
	int $0x80
	movb $1, %al # eax = 1, sys_exit
	int $0x80
```

這裡需要傳入 4 個參數，即讓棧彈出的第一個值，也就是參數個數賦予 `eax`，也就是：`hello 5 4 1`。

難道我們只能把該代碼優化到 10 個字節？

巧合地是，當偶然改成這樣的情況下，該代碼還能正常返回。

```
.global _start
_start:
	popl %eax	# eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
	popl %ecx	# argv[0], 字符串
	movb $5, %dl	# 設置字符串長度
	int $0x80
	loop _start     # 觸發系統退出
```

**注**：上面我們使用了 `loop` 指令而不是 `jmp` 指令，因為 `jmp _start` 產生的代碼更長，而 `loop _start` 指令只有兩個字節。

這裡相當於刪除了 `movb $1, %al`，最後我們獲得了 8 個字節。但是這裡為什麼能夠工作呢？

經過分析 `arch/x86/ia32/ia32entry.S`，我們發現當系統調用號無效時（超過系統調用入口個數），內核為了健壯考慮，必須要處理這類異常，並通過 `ia32_badsys` 讓系統調用正常返回。

這個可以這樣驗證：

```
.global _start
_start:
	popl %eax    # argc, eax = 4, 設置系統調用號, sys_write(fd, addr, len) : ebx, ecx, edx
	popl %ecx    # argv[0], 文件名
	mov $5, %dl  # argv[1]，字符串長度
	int $0x80
	mov $0xffffffda, %eax  # 設置一個非法調用號用於退出
	int $0x80
```

那最後的結果是，我們產生了一個可以正常打印字符串，大小隻有 45 字節的 `Elf` 文件，最終的結果如下：

```
# hello.s
#
# $ as --32 -o hello.o hello.s
# $ ld -melf_i386 --oformat=binary -o hello hello.o
# $ export PATH=./:$PATH
# $ hello 0 0 0
# hello
#

	.file "hello.s"
	.global _start, _load
	.equ   LOAD_ADDR, 0x00010000   # Page aligned load addr, here 64k
	.equ   E_ENTRY, LOAD_ADDR + (_start - _load)
	.equ   P_MEM_SZ, E_ENTRY
	.equ   P_FILE_SZ, P_MEM_SZ

_load:
	.byte  0x7F
	.ascii "ELF"              # e_ident, Magic Number
	.long  1                                      # p_type, loadable seg
	.long  0                                      # p_offset
	.long  LOAD_ADDR                              # p_vaddr
	.word  2                  # e_type, exec  # p_paddr
	.word  3                  # e_machine, Intel 386 target
	.long  P_FILE_SZ          # e_version     # p_filesz
	.long  E_ENTRY            # e_entry       # p_memsz
	.long  4                  # e_phoff       # p_flags, read(exec)
	.text
_start:
	popl   %eax    # argc     # e_shoff       # p_align
	               # 4 args, eax = 4, sys_write(fd, addr, len) : ebx, ecx, edx
	               # set 2nd eax = random addr to trigger bad syscall for exit
	popl   %ecx    # argv[0]
	mov    $5, %dl # str len  # e_flags
	int    $0x80
	loop   _start  # loop to popup a random addr as a bad syscall number
	.word  0x34               # e_ehsize = 52
	.word  0x20               # e_phentsize = 32
	.byte  1                  # e_phnum = 1, remove trailing 7 bytes with 0 value
	                          # e_shentsize
	                          # e_shnum
	                          # e_shstrndx
```

效果如下：

```
$ as --32 -o hello.o hello.s
$ ld -melf_i386 -o hello hello.o --oformat=binary
$ export PATH=./:$PATH
$ hello 0 0 0
hello
$ wc -c hello
45 hello
```

到這裡，我們獲得了史上最小的可以打印字符串的 `Elf` 文件，是的，只有 45 個字節。

<span id="toc_3928_6176_24"></span>
## 小結

到這裡，關於可執行文件的討論暫且結束，最後來一段小小的總結，那就是我們設法去減少可執行文件大小的意義？

實際上，通過這樣一個討論深入到了很多技術的細節，包括可執行文件的格式、目標代碼鏈接的過程、 Linux 下彙編語言開發等。與此同時，可執行文件大小的減少本身對嵌入式系統非常有用，如果刪除那些對程序運行沒有影響的節區和節區表將減少目標系統的大小，適應嵌入式系統資源受限的需求。除此之外，動態連接庫中的很多函數可能不會被使用到，因此也可以通過某種方式剔除 [\[8\]][8]，[\[10\]][10] 。

或許，你還會發現更多有趣的意義，歡迎給我發送郵件，一起討論。

<span id="toc_3928_6176_25"></span>
## 參考資料

- [A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux][1]
- [UNIX/LINUX 平臺可執行文件格式分析][2]
- [C/C++ 程序編譯步驟詳解][3]
- [The Linux GCC HOW TO][4]
- [ELF: From The Programmer's Perspective][5]
- [Understanding ELF using readelf and objdump][6]
- [Dissecting shared libraries][7]
- [嵌入式 Linux 小型化技術][8]
- [Linux 彙編語言開發指南][9]
- [Library Optimizer][10]
- ELF file format and ABI：[\[1\]][11]，[\[2\]][12]，[\[3\]][13]，[\[4\]][14]
- [i386 指令編碼表][15]

 [1]: http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html
 [2]: http://blog.chinaunix.net/u/19881/showart_215242.html
 [3]: http://www.xxlinux.com/linux/article/development/soft/20070424/8267.html
 [4]: http://www.faqs.org/docs/Linux-HOWTO/GCC-HOWTO.html
 [5]: http://linux.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html
 [6]: http://www.linuxforums.org/misc/understanding_elf_using_readelf_and_objdump.html
 [7]: http://www.ibm.com/developerworks/linux/library/l-shlibs.html
 [8]: http://www.gexin.com.cn/UploadFile/document2008119102415.pdf
 [9]: http://www.ibm.com/developerworks/cn/linux/l-assembly/index.html
 [10]: http://sourceforge.net/projects/libraryopt
 [11]: http://www.x86.org/ftp/manuals/tools/elf.pdf
 [12]: http://www.muppetlabs.com/~breadbox/software/ELF.txt
 [13]: http://162.105.203.48/web/gaikuang/submission/TN05.ELF.Format.Summary.pdf
 [14]: http://www.xfocus.net/articles/200105/174.html
 [15]: http://sparksandflames.com/files/x86InstructionChart.html
