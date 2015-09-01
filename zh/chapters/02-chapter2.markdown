# Gcc 編譯的背後

-    [前言](#toc_27212_14734_1)
-    [預處理](#toc_27212_14734_2)
    -    [簡述](#toc_27212_14734_3)
    -    [打印出預處理之後的結果](#toc_27212_14734_4)
    -    [在命令行定義宏](#toc_27212_14734_5)
-    [編譯（翻譯）](#toc_27212_14734_6)
    -    [簡述](#toc_27212_14734_7)
    -    [語法檢查](#toc_27212_14734_8)
    -    [編譯器優化](#toc_27212_14734_9)
    -    [生成彙編語言文件](#toc_27212_14734_10)
-    [彙編](#toc_27212_14734_11)
    -    [簡述](#toc_27212_14734_12)
    -    [生成目標代碼](#toc_27212_14734_13)
    -    [ELF 文件初次接觸](#toc_27212_14734_14)
    -    [ELF 文件的結構](#toc_27212_14734_15)
    -    [三種不同類型 ELF 文件比較](#toc_27212_14734_16)
    -    [ELF 主體：節區](#toc_27212_14734_17)
    -    [彙編語言文件中的節區表述](#toc_27212_14734_18)
-    [鏈接](#toc_27212_14734_19)
    -    [簡述](#toc_27212_14734_20)
    -    [可執行文件的段：節區重排](#toc_27212_14734_21)
    -    [鏈接背後的故事](#toc_27212_14734_22)
    -    [用 ld 完成鏈接過程](#toc_27212_14734_23)
    -    [C++ 構造與析構：crtbegin.o 和 crtend.o](#toc_27212_14734_24)
    -    [初始化與退出清理：crti.o 和 crtn.o](#toc_27212_14734_25)
    -    [C 語言程序真正的入口](#toc_27212_14734_26)
    -    [鏈接腳本初次接觸](#toc_27212_14734_27)
-    [參考資料](#toc_27212_14734_28)


<span id="toc_27212_14734_1"></span>
## 前言

平時在 Linux 下寫代碼，直接用 `gcc -o out in.c` 就把代碼編譯好了，但是這背後到底做了什麼呢？

如果學習過《編譯原理》則不難理解，一般高級語言程序編譯的過程莫過於：預處理、編譯、彙編、鏈接。

`gcc` 在後臺實際上也經歷了這幾個過程，可以通過 `-v` 參數查看它的編譯細節，如果想看某個具體的編譯過程，則可以分別使用 `-E`，`-S`，`-c` 和 `-O`，對應的後臺工具則分別為 `cpp`，`cc1`，`as`，`ld`。

下面將逐步分析這幾個過程以及相關的內容，諸如語法檢查、代碼調試、彙編語言等。

<span id="toc_27212_14734_2"></span>
## 預處理

<span id="toc_27212_14734_3"></span>
### 簡述

預處理是 C 語言程序從源代碼變成可執行程序的第一步，主要是 C 語言編譯器對各種預處理命令進行處理，包括頭文件的包含、宏定義的擴展、條件編譯的選擇等。

以前沒怎麼“深入”預處理，腦子對這些東西總是很模糊，只記得在編譯的基本過程（詞法分析、語法分析）之前還需要對源代碼中的宏定義、文件包含、條件編譯等命令進行處理。這三類的指令很常見，主要有 `#define`，`#include`和 `#ifdef ... #endif`，要特別地注意它們的用法。

`#define` 除了可以獨立使用以便靈活設置一些參數外，還常常和 `#ifdef ... #endif` 結合使用，以便靈活地控制代碼塊的編譯與否，也可以用來避免同一個頭文件的多次包含。關於 `#include` 貌似比較簡單，通過 `man` 找到某個函數的頭文件，複製進去，加上 `<>` 就好。這裡雖然只關心一些技巧，不過預處理還是隱藏著很多潛在的陷阱（可參考《C Traps & Pitfalls》）也是需要注意的。下面僅介紹和預處理相關的幾個簡單內容。

<span id="toc_27212_14734_4"></span>
### 打印出預處理之後的結果

```
$ gcc -E hello.c
```

這樣就可以看到源代碼中的各種預處理命令是如何被解釋的，從而方便理解和查錯。

實際上 `gcc` 在這裡調用了 `cpp`（雖然通過 `gcc -v` 僅看到 `cc1`)，`cpp` 即 The C Preprocessor，主要用來預處理宏定義、文件包含、條件編譯等。下面介紹它的一個比較重要的選項 `-D`。

<span id="toc_27212_14734_5"></span>
### 在命令行定義宏

```
$ gcc -Dmacro hello.c
```

這個等同於在文件的開頭定義宏，即 `#define macro`，但是在命令行定義更靈活。例如，在源代碼中有這些語句。

```
#ifdef DEBUG
printf("this code is for debugging\n");
#endif
```

如果編譯時加上 `-DDEBUG` 選項，那麼編譯器就會把 `printf` 所在的行編譯進目標代碼，從而方便地跟蹤該位置的某些程序狀態。這樣 `-DDEBUG` 就可以當作一個調試開關，編譯時加上它就可以用來打印調試信息，發佈時則可以通過去掉該編譯選項把調試信息去掉。

<span id="toc_27212_14734_6"></span>
## 編譯（翻譯）

<span id="toc_27212_14734_7"></span>
### 簡述

編譯之前，C 語言編譯器會進行詞法分析、語法分析，接著會把源代碼翻譯成中間語言，即彙編語言。如果想看到這個中間結果，可以用 `gcc -S`。需要提到的是，諸如 Shell 等解釋語言也會經歷一個詞法分析和語法分析的階段，不過之後並不會進行“翻譯”，而是“解釋”，邊解釋邊執行。

把源代碼翻譯成彙編語言，實際上是編譯的整個過程中的第一個階段，之後的階段和彙編語言的開發過程沒有什麼區別。這個階段涉及到對源代碼的詞法分析、語法檢查（通過 `-std` 指定遵循哪個標準），並根據優化（`-O`）要求進行翻譯成彙編語言的動作。

<span id="toc_27212_14734_8"></span>
### 語法檢查

如果僅僅希望進行語法檢查，可以用 `gcc` 的 `-fsyntax-only` 選項；如果為了使代碼有比較好的可移植性，避免使用 `gcc` 的一些擴展特性，可以結合 `-std` 和 `-pedantic`（或者 `-pedantic-erros` ）選項讓源代碼遵循某個 C 語言標準的語法。這裡演示一個簡單的例子：

```
$ cat hello.c
#include <stdio.h>
int main()
{
	printf("hello, world\n")
	return 0;
}
$ gcc -fsyntax-only hello.c
hello.c: In function ‘main’:
hello.c:5: error: expected ‘;’ before ‘return’
$ vim hello.c
$ cat hello.c
#include <stdio.h>
int main()
{
        printf("hello, world\n");
        int i;
        return 0;
}
$ gcc -std=c89 -pedantic-errors hello.c    #默認情況下，gcc是允許在程序中間聲明變量的，但是turboc就不支持
hello.c: In function ‘main’:
hello.c:5: error: ISO C90 forbids mixed declarations and code
```

語法錯誤是程序開發過程中難以避免的錯誤（人的大腦在很多情況下都容易開小差），不過編譯器往往能夠通過語法檢查快速發現這些錯誤，並準確地告知語法錯誤的大概位置。因此，作為開發人員，要做的事情不是“恐慌”（不知所措），而是認真閱讀編譯器的提示，根據平時積累的經驗（最好總結一份常見語法錯誤索引，很多資料都提供了常見語法錯誤列表，如《C Traps&Pitfalls》和編輯器提供的語法檢查功能（語法加亮、括號匹配提示等）快速定位語法出錯的位置並進行修改。

<span id="toc_27212_14734_9"></span>
### 編譯器優化

語法檢查之後就是翻譯動作，`gcc` 提供了一個優化選項 `-O`，以便根據不同的運行平臺和用戶要求產生經過優化的彙編代碼。例如，

```
$ gcc -o hello hello.c         # 採用默認選項，不優化
$ gcc -O2 -o hello2 hello.c    # 優化等次是2
$ gcc -Os -o hellos hello.c    # 優化目標代碼的大小
$ ls -S hello hello2 hellos    # 可以看到，hellos 比較小, hello2 比較大
hello2  hello  hellos
$ time ./hello
hello, world

real    0m0.001s
user    0m0.000s
sys     0m0.000s
$ time ./hello2     # 可能是代碼比較少的緣故，執行效率看上去不是很明顯
hello, world

real    0m0.001s
user    0m0.000s
sys     0m0.000s

$ time ./hellos     # 雖然目標代碼小了，但是執行效率慢了些
hello, world

real    0m0.002s
user    0m0.000s
sys     0m0.000s
```

根據上面的簡單演示，可以看出 `gcc` 有很多不同的優化選項，主要看用戶的需求了，目標代碼的大小和效率之間貌似存在一個“糾纏”，需要開發人員自己權衡。

<span id="toc_27212_14734_10"></span>
### 生成彙編語言文件

下面通過 `-S` 選項來看看編譯出來的中間結果：彙編語言，還是以之前那個 `hello.c` 為例。

```
$ gcc -S hello.c  # 默認輸出是hello.s，可自己指定，輸出到屏幕`-o -`，輸出到其他文件`-o file`
$ cat hello.s
cat hello.s
        .file   "hello.c"
        .section        .rodata
.LC0:
        .string "hello, world"
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
        movl    $0, %eax
        addl    $4, %esp
        popl    %ecx
        popl    %ebp
        leal    -4(%ecx), %esp
        ret
        .size   main, .-main
        .ident  "GCC: (GNU) 4.1.3 20070929 (prerelease) (Ubuntu 4.1.2-16ubuntu2)"
        .section        .note.GNU-stack,"",@progbits
```

不知道看出來沒？和課堂裡學的 intel 的彙編語法不太一樣，這裡用的是 `AT&T` 語法格式。如果想學習 Linux 下的彙編語言開發，下一節開始的所有章節基本上覆蓋了 Linux 下彙編語言開發的一般過程，不過這裡不介紹彙編語言語法。

在學習後面的章節之前，建議自學舊金山大學的微機編程課程 CS630，該課深入介紹了 Linux/X86 平臺下的 `AT&T` 彙編語言開發。如果想在 `Qemu` 上做這個課程裡的實驗，可以閱讀本文作者寫的 [CS630: Linux 下通過 Qemu 學習 X86 AT&T 彙編語言](http://www.tinylab.org/cs630-qemu/)。

需要補充的是，在寫 C 語言代碼時，如果能夠對編譯器比較熟悉（工作原理和一些細節）的話，可能會很有幫助。包括這裡的優化選項（有些優化選項可能在彙編時採用）和可能的優化措施，例如字節對齊、條件分支語句裁減（刪除一些明顯分支）等。

<span id="toc_27212_14734_11"></span>
## 彙編

<span id="toc_27212_14734_12"></span>
### 簡述

彙編實際上還是翻譯過程，只不過把作為中間結果的彙編代碼翻譯成了機器代碼，即目標代碼，不過它還不可以運行。如果要產生這一中間結果，可用 `gcc -c`，當然，也可通過 `as` 命令處理彙編語言源文件來產生。

彙編是把彙編語言翻譯成目標代碼的過程，如果有在 Windows 下學習過彙編語言開發，大家應該比較熟悉 `nasm` 彙編工具(支持 Intel 格式的彙編語言)，不過這裡主要用 `as` 彙編工具來彙編 `AT&T` 格式的彙編語言，因為 `gcc` 產生的中間代碼就是 `AT&T` 格式的。

<span id="toc_27212_14734_13"></span>
### 生成目標代碼

下面來演示分別通過 `gcc -c` 選項和 `as` 來產生目標代碼。

```
$ file hello.s
hello.s: ASCII assembler program text
$ gcc -c hello.s   #用gcc把彙編語言編譯成目標代碼
$ file hello.o     #file命令用來查看文件類型，目標代碼可重定位的(relocatable)，
                   #需要通過ld進行進一步鏈接成可執行程序(executable)和共享庫(shared)
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
$ as -o hello.o hello.s        #用as把彙編語言編譯成目標代碼
$ file hello.o
hello.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
```

`gcc` 和 `as` 默認產生的目標代碼都是 ELF 格式的，因此這裡主要討論ELF格式的目標代碼（如果有時間再回顧一下 `a.out` 和 `coff` 格式，當然也可以先了解一下，並結合 `objcopy` 來轉換它們，比較異同)。

<span id="toc_27212_14734_14"></span>
### ELF 文件初次接觸

目標代碼不再是普通的文本格式，無法直接通過文本編輯器瀏覽，需要一些專門的工具。如果想了解更多目標代碼的細節，區分 `relocatable`（可重定位）、`executable`（可執行）、`shared libarary`（共享庫）的不同，我們得設法瞭解目標代碼的組織方式和相關的閱讀和分析工具。下面主要介紹這部分內容。

> BFD is a package which allows applications to use the same routines to
operate on object files whatever the object file format. A new object file
format can be supported simply by creating a new BFD back end and adding it to
the library.

binutils（GNU Binary Utilities）的很多工具都採用這個庫來操作目標文件，這類工具有 `objdump`，`objcopy`，`nm`，`strip` 等（當然，我們也可以利用它。如果深入瞭解ELF格式，那麼通過它來分析和編寫 Virus 程序將會更加方便），不過另外一款非常優秀的分析工具 `readelf` 並不是基於這個庫，所以也應該可以直接用 `elf.h` 頭文件中定義的相關結構來操作 ELF 文件。

下面將通過這些輔助工具（主要是 `readelf` 和 `objdump`），結合 ELF 手冊來分析它們。將依次介紹 ELF 文件的結構和三種不同類型 ELF 文件的區別。

<span id="toc_27212_14734_15"></span>
### ELF 文件的結構

```
ELF Header(ELF文件頭)
Program Headers Table(程序頭表，實際上叫段表好一些，用於描述可執行文件和可共享庫)
Section 1
Section 2
Section 3
...
Section Headers Table(節區頭部表，用於鏈接可重定位文件成可執行文件或共享庫)
```

對於可重定位文件，程序頭是可選的，而對於可執行文件和共享庫文件（動態鏈接庫），節區表則是可選的。可以分別通過 `readelf` 文件的 `-h`，`-l` 和 `-S` 參數查看 ELF 文件頭（ELF Header）、程序頭部表（Program Headers Table，段表）和節區表（Section Headers Table）。

文件頭說明了文件的類型，大小，運行平臺，節區數目等。

<span id="toc_27212_14734_16"></span>
### 三種不同類型 ELF 文件比較

先來通過文件頭看看不同ELF的類型。為了說明問題，先來幾段代碼吧。

```
/* myprintf.c */
#include <stdio.h>

void myprintf(void)
{
	printf("hello, world!\n");
}
```

```
/* test.h -- myprintf function declaration */

#ifndef _TEST_H_
#define _TEST_H_

void myprintf(void);

#endif
```

```
/* test.c */
#include "test.h"

int main()
{
	myprintf();
	return 0;
}
```


下面通過這幾段代碼來演示通過 `readelf -h` 參數查看 ELF 的不同類型。期間將演示如何創建動態鏈接庫（即可共享文件）、靜態鏈接庫，並比較它們的異同。


編譯產生兩個目標文件 `myprintf.o` 和 `test.o`，它們都是可重定位文件（REL）：

```
$ gcc -c myprintf.c test.c
$ readelf -h test.o | grep Type
  Type:                              REL (Relocatable file)
$ readelf -h myprintf.o | grep Type
  Type:                              REL (Relocatable file)
```

根據目標代碼鏈接產生可執行文件，這裡的文件類型是可執行的(EXEC)：

```
$ gcc -o test myprintf.o test.o
$ readelf -h test | grep Type
  Type:                              EXEC (Executable file)
```

用 `ar` 命令創建一個靜態鏈接庫，靜態鏈接庫也是可重定位文件（REL）：

```
$ ar rcsv libmyprintf.a myprintf.o
$ readelf -h libmyprintf.a | grep Type
  Type:                              REL (Relocatable file)
```

可見，靜態鏈接庫和可重定位文件類型一樣，它們之間唯一不同是前者可以是多個可重定位文件的“集合”。

靜態鏈接庫可直接鏈接（只需庫名，不要前面的 `lib`），也可用 `-l` 參數，`-L` 指定庫搜索路徑。

```
$ gcc -o test test.o -lmyprintf -L./
```

編譯產生動態鏈接庫，並支持 `major` 和 `minor` 版本號，動態鏈接庫類型為 `DYN`：

```
$ gcc -Wall myprintf.o -shared -Wl,-soname,libmyprintf.so.0 -o libmyprintf.so.0.0
$ ln -sf libmyprintf.so.0.0 libmyprintf.so.0
$ ln -sf libmyprintf.so.0 libmyprintf.so
$ readelf -h libmyprintf.so | grep Type
  Type:                              DYN (Shared object file)
```

動態鏈接庫編譯時和靜態鏈接庫類似：

```
$ gcc -o test test.o -lmyprintf -L./
```

但是執行時需要指定動態鏈接庫的搜索路徑，把 `LD_LIBRARY_PATH` 設為當前目錄，指定 `test` 運行時的動態鏈接庫搜索路徑：

```
$ LD_LIBRARY_PATH=./ ./test
$ gcc -static -o test test.o -lmyprintf -L./
```

在不指定 `-static` 時會優先使用動態鏈接庫，指定時則阻止使用動態鏈接庫，這時會把所有靜態鏈接庫文件加入到可執行文件中，使得執行文件很大，而且加載到內存以後會浪費內存空間，因此不建議這麼做。

經過上面的演示基本可以看出它們之間的不同：

- 可重定位文件本身不可以運行，僅僅是作為可執行文件、靜態鏈接庫（也是可重定位文件）、動態鏈接庫的 “組件”。
- 靜態鏈接庫和動態鏈接庫本身也不可以執行，作為可執行文件的“組件”，它們兩者也不同，前者也是可重定位文件（只不過可能是多個可重定位文件的集合），並且在鏈接時加入到可執行文件中去。
- 而動態鏈接庫在鏈接時，庫文件本身並沒有添加到可執行文件中，只是在可執行文件中加入了該庫的名字等信息，以便在可執行文件運行過程中引用庫中的函數時由動態鏈接器去查找相關函數的地址，並調用它們。

從這個意義上說，動態鏈接庫本身也具有可重定位的特徵，含有可重定位的信息。對於什麼是重定位？如何進行靜態符號和動態符號的重定位，我們將在鏈接部分和[《動態符號鏈接的細節》][100]一節介紹。

[100]: 02-chapter4.markdown

<span id="toc_27212_14734_17"></span>
### ELF 主體：節區

下面來看看 ELF 文件的主體內容：節區（Section)。

ELF 文件具有很大的靈活性，它通過文件頭組織整個文件的總體結構，通過節區表 (Section Headers Table）和程序頭（Program Headers Table 或者叫段表）來分別描述可重定位文件和可執行文件。但不管是哪種類型，它們都需要它們的主體，即各種節區。

在可重定位文件中，節區表描述的就是各種節區本身；而在可執行文件中，程序頭描述的是由各個節區組成的段（Segment），以便程序運行時動態裝載器知道如何對它們進行內存映像，從而方便程序加載和運行。

下面先來看看一些常見的節區，而關於這些節區（Section）如何通過重定位構成不同的段（Segments），以及有哪些常規的段，我們將在鏈接部分進一步介紹。

可以通過 `readelf -S` 查看 ELF 的節區。（建議一邊操作一邊看文檔，以便加深對 ELF 文件結構的理解）先來看看可重定位文件的節區信息，通過節區表來查看：

默認編譯好 `myprintf.c`，將產生一個可重定位的文件 `myprintf.o`，這裡通過 `myprintf.o` 的節區表查看節區信息。

```
$ gcc -c myprintf.c
$ readelf -S myprintf.o
There are 11 section headers, starting at offset 0xc0:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000034 000018 00  AX  0   0  4
  [ 2] .rel.text         REL             00000000 000334 000010 08      9   1  4
  [ 3] .data             PROGBITS        00000000 00004c 000000 00  WA  0   0  4
  [ 4] .bss              NOBITS          00000000 00004c 000000 00  WA  0   0  4
  [ 5] .rodata           PROGBITS        00000000 00004c 00000e 00   A  0   0  1
  [ 6] .comment          PROGBITS        00000000 00005a 000012 00      0   0  1
  [ 7] .note.GNU-stack   PROGBITS        00000000 00006c 000000 00      0   0  1
  [ 8] .shstrtab         STRTAB          00000000 00006c 000051 00      0   0  1
  [ 9] .symtab           SYMTAB          00000000 000278 0000a0 10     10   8  4
  [10] .strtab           STRTAB          00000000 000318 00001a 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

用 `objdump -d` 可看反編譯結果，用 `-j` 選項可指定需要查看的節區：

```
$ objdump -d -j .text   myprintf.o
myprintf.o:     file format elf32-i386

Disassembly of section .text:

00000000 <myprintf>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   83 ec 08                sub    $0x8,%esp
   6:   83 ec 0c                sub    $0xc,%esp
   9:   68 00 00 00 00          push   $0x0
   e:   e8 fc ff ff ff          call   f <myprintf+0xf>
  13:   83 c4 10                add    $0x10,%esp
  16:   c9                      leave
  17:   c3                      ret
```

用 `-r` 選項可以看到有關重定位的信息，這裡有兩部分需要重定位：

```
$ readelf -r myprintf.o

Relocation section '.rel.text' at offset 0x334 contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0000000a  00000501 R_386_32          00000000   .rodata
0000000f  00000902 R_386_PC32        00000000   puts
```

`.rodata` 節區包含只讀數據，即我們要打印的 `hello, world!`

```
$ readelf -x .rodata myprintf.o

Hex dump of section '.rodata':
  0x00000000 68656c6c 6f2c2077 6f726c64 2100     hello, world!.
```

沒有找到 `.data` 節區, 它應該包含一些初始化的數據：

```
$ readelf -x .data myprintf.o

Section '.data' has no data to dump.
```

也沒有 `.bss` 節區，它應該包含一些未初始化的數據，程序默認初始為 0：

```
$ readelf -x .bss       myprintf.o

Section '.bss' has no data to dump.
```

`.comment` 是一些註釋，可以看到是是 `Gcc` 的版本信息

```
$ readelf -x .comment myprintf.o

Hex dump of section '.comment':
  0x00000000 00474343 3a202847 4e552920 342e312e .GCC: (GNU) 4.1.
  0x00000010 3200                                2.
```

`.note.GNU-stack` 這個節區也沒有內容：

```
$ readelf -x .note.GNU-stack myprintf.o

Section '.note.GNU-stack' has no data to dump.
```

`.shstrtab` 包括所有節區的名字：

```
$ readelf -x .shstrtab myprintf.o

Hex dump of section '.shstrtab':
  0x00000000 002e7379 6d746162 002e7374 72746162 ..symtab..strtab
  0x00000010 002e7368 73747274 6162002e 72656c2e ..shstrtab..rel.
  0x00000020 74657874 002e6461 7461002e 62737300 text..data..bss.
  0x00000030 2e726f64 61746100 2e636f6d 6d656e74 .rodata..comment
  0x00000040 002e6e6f 74652e47 4e552d73 7461636b ..note.GNU-stack
  0x00000050 00                                  .
```

符號表 `.symtab` 包括所有用到的相關符號信息，如函數名、變量名，可用 `readelf` 查看：

```
$ readelf -symtab myprintf.o

Symbol table '.symtab' contains 10 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FILE    LOCAL  DEFAULT  ABS myprintf.c
     2: 00000000     0 SECTION LOCAL  DEFAULT    1
     3: 00000000     0 SECTION LOCAL  DEFAULT    3
     4: 00000000     0 SECTION LOCAL  DEFAULT    4
     5: 00000000     0 SECTION LOCAL  DEFAULT    5
     6: 00000000     0 SECTION LOCAL  DEFAULT    7
     7: 00000000     0 SECTION LOCAL  DEFAULT    6
     8: 00000000    24 FUNC    GLOBAL DEFAULT    1 myprintf
     9: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND puts
```

字符串表 `.strtab` 包含用到的字符串，包括文件名、函數名、變量名等：

```
$ readelf -x .strtab myprintf.o

Hex dump of section '.strtab':
  0x00000000 006d7970 72696e74 662e6300 6d797072 .myprintf.c.mypr
  0x00000010 696e7466 00707574 7300              intf.puts.
```

從上表可以看出，對於可重定位文件，會包含這些基本節區 `.text`, `.rel.text`, `.data`, `.bss`, `.rodata`, `.comment`, `.note.GNU-stack`, `.shstrtab`, `.symtab` 和 `.strtab`。

<span id="toc_27212_14734_18"></span>
### 彙編語言文件中的節區表述

為了進一步理解這些節區和源代碼的關係，這裡來看一看 `myprintf.c` 產生的彙編代碼。

```
$ gcc -S myprintf.c
$ cat myprintf.s
        .file   "myprintf.c"
        .section        .rodata
.LC0:
        .string "hello, world!"
        .text
.globl myprintf
        .type   myprintf, @function
myprintf:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $8, %esp
        subl    $12, %esp
        pushl   $.LC0
        call    puts
        addl    $16, %esp
        leave
        ret
        .size   myprintf, .-myprintf
        .ident  "GCC: (GNU) 4.1.2"
        .section        .note.GNU-stack,"",@progbits
```

是不是可以從中看出可重定位文件中的那些節區和彙編語言代碼之間的關係？在上面的可重定位文件，可以看到有一個可重定位的節區，即 `.rel.text`，它標記了兩個需要重定位的項，`.rodata` 和 `puts`。這個節區將告訴編譯器這兩個信息在鏈接或者動態鏈接的過程中需要重定位， 具體如何重定位？將根據重定位項的類型，比如上面的 `R_386_32` 和 `R_386_PC32`。

到這裡，對可重定位文件應該有了一個基本的瞭解，下面將介紹什麼是可重定位，可重定位文件到底是如何被鏈接生成可執行文件和動態鏈接庫的，這個過程除了進行一些符號的重定位外，還進行了哪些工作呢？

<span id="toc_27212_14734_19"></span>
## 鏈接

<span id="toc_27212_14734_20"></span>
### 簡述

重定位是將符號引用與符號定義進行鏈接的過程。因此鏈接是處理可重定位文件，把它們的各種符號引用和符號定義轉換為可執行文件中的合適信息（一般是虛擬內存地址）的過程。

鏈接又分為靜態鏈接和動態鏈接，前者是程序開發階段程序員用 `ld`（`gcc` 實際上在後臺調用了 `ld`）靜態鏈接器手動鏈接的過程，而動態鏈接則是程序運行期間系統調用動態鏈接器（`ld-linux.so`）自動鏈接的過程。

比如，如果鏈接到可執行文件中的是靜態鏈接庫 `libmyprintf.a`，那麼 `.rodata` 節區在鏈接後需要被重定位到一個絕對的虛擬內存地址，以便程序運行時能夠正確訪問該節區中的字符串信息。而對於 `puts` 函數，因為它是動態鏈接庫 `libc.so` 中定義的函數，所以會在程序運行時通過動態符號鏈接找出 `puts` 函數在內存中的地址，以便程序調用該函數。在這裡主要討論靜態鏈接過程，動態鏈接過程見[《動態符號鏈接的細節》][100]。

靜態鏈接過程主要是把可重定位文件依次讀入，分析各個文件的文件頭，進而依次讀入各個文件的節區，並計算各個節區的虛擬內存位置，對一些需要重定位的符號進行處理，設定它們的虛擬內存地址等，並最終產生一個可執行文件或者是動態鏈接庫。這個鏈接過程是通過 `ld` 來完成的，`ld` 在鏈接時使用了一個鏈接腳本（`linker script`），該鏈接腳本處理鏈接的具體細節。

由於靜態符號鏈接過程非常複雜，特別是計算符號地址的過程，考慮到時間關係，相關細節請參考 ELF 手冊。這裡主要介紹可重定位文件中的節區（節區表描述的）和可執行文件中段（程序頭描述的）的對應關係以及 `gcc` 編譯時採用的一些默認鏈接選項。

<span id="toc_27212_14734_21"></span>
### 可執行文件的段：節區重排

下面先來看看可執行文件的節區信息，通過程序頭（段表）來查看，為了比較，先把 `test.o` 的節區表也列出：

```
$ readelf -S test.o
There are 10 section headers, starting at offset 0xb4:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000034 000024 00  AX  0   0  4
  [ 2] .rel.text         REL             00000000 0002ec 000008 08      8   1  4
  [ 3] .data             PROGBITS        00000000 000058 000000 00  WA  0   0  4
  [ 4] .bss              NOBITS          00000000 000058 000000 00  WA  0   0  4
  [ 5] .comment          PROGBITS        00000000 000058 000012 00      0   0  1
  [ 6] .note.GNU-stack   PROGBITS        00000000 00006a 000000 00      0   0  1
  [ 7] .shstrtab         STRTAB          00000000 00006a 000049 00      0   0  1
  [ 8] .symtab           SYMTAB          00000000 000244 000090 10      9   7  4
  [ 9] .strtab           STRTAB          00000000 0002d4 000016 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
$ gcc -o test test.o myprintf.o
$ readelf -l test

Elf file type is EXEC (Executable file)
Entry point 0x80482b0
There are 7 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x000e0 0x000e0 R E 0x4
  INTERP         0x000114 0x08048114 0x08048114 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x0047c 0x0047c R E 0x1000
  LOAD           0x00047c 0x0804947c 0x0804947c 0x00104 0x00108 RW  0x1000
  DYNAMIC        0x000490 0x08049490 0x08049490 0x000c8 0x000c8 RW  0x4
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
```

可發現，`test` 和 `test.o`，`myprintf.o` 相比，多了很多節區，如 `.interp` 和 `.init` 等。另外，上表也給出了可執行文件的如下幾個段（Segment）：

- `PHDR`: 給出了程序表自身的大小和位置，不能出現一次以上。
- `INTERP`: 因為程序中調用了 `puts`（在動態鏈接庫中定義），使用了動態鏈接庫，因此需要動態裝載器／鏈接器（`ld-linux.so`）
- `LOAD`: 包括程序的指令，`.text` 等節區都映射在該段，只讀（R）
- `LOAD`: 包括程序的數據，`.data`,`.bss` 等節區都映射在該段，可讀寫（RW）
- `DYNAMIC`: 動態鏈接相關的信息，比如包含有引用的動態鏈接庫名字等信息
- `NOTE`: 給出一些附加信息的位置和大小
- `GNU_STACK`: 這裡為空，應該是和GNU相關的一些信息

這裡的段可能包括之前的一個或者多個節區，也就是說經過鏈接之後原來的節區被重排了，並映射到了不同的段，這些段將告訴系統應該如何把它加載到內存中。

<span id="toc_27212_14734_22"></span>
### 鏈接背後的故事

從上表中，通過比較可執行文件 `test` 中擁有的節區和可重定位文件（`test.o` 和 `myprintf.o`）中擁有的節區後發現，鏈接之後多了一些之前沒有的節區，這些新的節區來自哪裡？它們的作用是什麼呢？先來通過 `gcc -v` 看看它的後臺鏈接過程。

把可重定位文件鏈接成可執行文件：

```
$ gcc -v -o test test.o myprintf.o
Reading specs from /usr/lib/gcc/i486-slackware-linux/4.1.2/specs
Target: i486-slackware-linux
Configured with: ../gcc-4.1.2/configure --prefix=/usr --enable-shared
--enable-languages=ada,c,c++,fortran,java,objc --enable-threads=posix
--enable-__cxa_atexit --disable-checking --with-gnu-ld --verbose
--with-arch=i486 --target=i486-slackware-linux --host=i486-slackware-linux
Thread model: posix
gcc version 4.1.2
 /usr/libexec/gcc/i486-slackware-linux/4.1.2/collect2 --eh-frame-hdr -m
elf_i386 -dynamic-linker /lib/ld-linux.so.2 -o test
/usr/lib/gcc/i486-slackware-linux/4.1.2/../../../crt1.o
/usr/lib/gcc/i486-slackware-linux/4.1.2/../../../crti.o
/usr/lib/gcc/i486-slackware-linux/4.1.2/crtbegin.o
-L/usr/lib/gcc/i486-slackware-linux/4.1.2
-L/usr/lib/gcc/i486-slackware-linux/4.1.2
-L/usr/lib/gcc/i486-slackware-linux/4.1.2/../../../../i486-slackware-linux/lib
-L/usr/lib/gcc/i486-slackware-linux/4.1.2/../../.. test.o myprintf.o -lgcc
--as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed
/usr/lib/gcc/i486-slackware-linux/4.1.2/crtend.o
/usr/lib/gcc/i486-slackware-linux/4.1.2/../../../crtn.o
```

從上述演示看出，`gcc` 在鏈接了我們自己的目標文件 `test.o` 和 `myprintf.o` 之外，還鏈接了 `crt1.o`，`crtbegin.o` 等額外的目標文件，難道那些新的節區就來自這些文件？

<span id="toc_27212_14734_23"></span>
### 用 ld 完成鏈接過程

另外 `gcc` 在進行了相關配置（`./configure`）後，調用了 `collect2`，卻並沒有調用 `ld`，通過查找 `gcc` 文檔中和 `collect2` 相關的部分發現 `collect2` 在後臺實際上還是去尋找 `ld` 命令的。為了理解 `gcc` 默認鏈接的後臺細節，這裡直接把 `collect2` 替換成 `ld`，並把一些路徑換成絕對路徑或者簡化，得到如下的 `ld` 命令以及執行的效果。

```
$ ld --eh-frame-hdr \
-m elf_i386 \
-dynamic-linker /lib/ld-linux.so.2 \
-o test \
/usr/lib/crt1.o /usr/lib/crti.o /usr/lib/gcc/i486-slackware-linux/4.1.2/crtbegin.o \
test.o myprintf.o \
-L/usr/lib/gcc/i486-slackware-linux/4.1.2 -L/usr/i486-slackware-linux/lib -L/usr/lib/ \
-lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed \
/usr/lib/gcc/i486-slackware-linux/4.1.2/crtend.o /usr/lib/crtn.o
$ ./test
hello, world!
```

不出所料，它完美地運行了。下面通過 `ld` 的手冊（`man ld`）來分析一下這幾個參數：

- `--eh-frame-hdr`

    要求創建一個 `.eh_frame_hdr` 節區(貌似目標文件test中並沒有這個節區，所以不關心它)。

- `-m elf_i386`

    這裡指定不同平臺上的鏈接腳本，可以通過 `--verbose` 命令查看腳本的具體內容，如 `ld -m elf_i386 --verbose`，它實際上被存放在一個文件中（`/usr/lib/ldscripts` 目錄下），我們可以去修改這個腳本，具體如何做？請參考 `ld` 的手冊。在後面我們將簡要提到鏈接腳本中是如何預定義變量的，以及這些預定義變量如何在我們的程序中使用。需要提到的是，如果不是交叉編譯，那麼無須指定該選項。

- -dynamic-linker /lib/ld-linux.so.2

    指定動態裝載器/鏈接器，即程序中的 `INTERP` 段中的內容。動態裝載器/鏈接器負責鏈接有可共享庫的可執行文件的裝載和動態符號鏈接。

- -o test

    指定輸出文件，即可執行文件名的名字

- /usr/lib/crt1.o /usr/lib/crti.o /usr/lib/gcc/i486-slackware-linux/4.1.2/crtbegin.o

    鏈接到 `test` 文件開頭的一些內容，這裡實際上就包含了 `.init` 等節區。`.init` 節區包含一些可執行代碼，在 `main` 函數之前被調用，以便進行一些初始化操作，在 C++ 中完成構造函數功能。

- test.o myprintf.o

    鏈接我們自己的可重定位文件

- `-L/usr/lib/gcc/i486-slackware-linux/4.1.2 -L/usr/i486-slackware-linux/lib -L/usr/lib/ \
  -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed`

    鏈接 `libgcc` 庫和 `libc` 庫，後者定義有我們需要的 `puts` 函數

- /usr/lib/gcc/i486-slackware-linux/4.1.2/crtend.o /usr/lib/crtn.o

    鏈接到 `test` 文件末尾的一些內容，這裡實際上包含了 `.fini` 等節區。`.fini` 節區包含了一些可執行代碼，在程序退出時被執行，作一些清理工作，在 C++ 中完成析構造函數功能。我們往往可以通過 `atexit` 來註冊那些需要在程序退出時才執行的函數。

<span id="toc_27212_14734_24"></span>
### C++構造與析構：crtbegin.o和crtend.o

對於 `crtbegin.o` 和 `crtend.o` 這兩個文件，貌似完全是用來支持 C++ 的構造和析構工作的，所以可以不鏈接到我們的可執行文件中，鏈接時把它們去掉看看，

```
$ ld -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -o test \
  /usr/lib/crt1.o /usr/lib/crti.o test.o myprintf.o \
  -L/usr/lib -lc /usr/lib/crtn.o    #後面發現不用鏈接libgcc，也不用--eh-frame-hdr參數
$ readelf -l test

Elf file type is EXEC (Executable file)
Entry point 0x80482b0
There are 7 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x000e0 0x000e0 R E 0x4
  INTERP         0x000114 0x08048114 0x08048114 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x003ea 0x003ea R E 0x1000
  LOAD           0x0003ec 0x080493ec 0x080493ec 0x000e8 0x000e8 RW  0x1000
  DYNAMIC        0x0003ec 0x080493ec 0x080493ec 0x000c8 0x000c8 RW  0x4
  NOTE           0x000128 0x08048128 0x08048128 0x00020 0x00020 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .hash .dynsym .dynstr .gnu.version .gnu.version_r
          .rel.dyn .rel.plt .init .plt .text .fini .rodata
   03     .dynamic .got .got.plt .data
   04     .dynamic
   05     .note.ABI-tag
   06
$ ./test
hello, world!
```

完全可以工作，而且發現 `.ctors`（保存著程序中全局構造函數的指針數組）, `.dtors`（保存著程序中全局析構函數的指針數組）,`.jcr`（未知）,`.eh_frame` 節區都沒有了，所以 `crtbegin.o` 和 `crtend.o` 應該包含了這些節區。

<span id="toc_27212_14734_25"></span>
### 初始化與退出清理：crti.o 和 crtn.o

而對於另外兩個文件 `crti.o` 和 `crtn.o`，通過 `readelf -S` 查看後發現它們都有 `.init` 和 `.fini` 節區，如果我們不需要讓程序進行一些初始化和清理工作呢？是不是就可以不鏈接這個兩個文件？試試看。

```
$ ld  -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -o test \
      /usr/lib/crt1.o test.o myprintf.o -L/usr/lib/ -lc
/usr/lib/libc_nonshared.a(elf-init.oS): In function `__libc_csu_init':
(.text+0x25): undefined reference to `_init'
```

貌似不行，竟然有人調用了 `__libc_csu_init` 函數，而這個函數引用了 `_init`。這兩個符號都在哪裡呢？

```
$ readelf -s /usr/lib/crt1.o | grep __libc_csu_init
    18: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND __libc_csu_init
$ readelf -s /usr/lib/crti.o | grep _init
    17: 00000000     0 FUNC    GLOBAL DEFAULT    5 _init
```

竟然是 `crt1.o` 調用了 `__libc_csu_init` 函數，而該函數卻引用了我們沒有鏈接的 `crti.o` 文件中定義的 `_init` 符號。這樣的話不鏈接 `crti.o` 和 `crtn.o` 文件就不成了羅？不對吧，要不乾脆不用 `crt1.o` 算了，看看 `gcc` 額外鏈接進去的最後一個文件 `crt1.o` 到底幹了個啥子？

```
$ ld  -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -o \
      test test.o myprintf.o -L/usr/lib/ -lc
ld: warning: cannot find entry symbol _start; defaulting to 00000000080481a4
```

這樣卻說沒有找到入口符號 `_start`，難道 `crt1.o` 中定義了這個符號？不過它給默認設置了一個地址，只是個警告，說明 `test` 已經生成，不管怎樣先運行看看再說。

```
$ ./test
hello, world!
Segmentation fault
```

貌似程序運行完了，不過結束時冒出個段錯誤？可能是程序結束時有問題，用 `gdb` 調試看看：

```
$ gcc -g -c test.c myprintf.c #產生目標代碼, 非交叉編譯，不指定-m也可鏈接，所以下面可去掉-m
$ ld -dynamic-linker /lib/ld-linux.so.2 -o test \
     test.o myprintf.o -L/usr/lib -lc
ld: warning: cannot find entry symbol _start; defaulting to 00000000080481d8
$ ./test
hello, world!
Segmentation fault
$ gdb ./test
...
(gdb) l
1       #include "test.h"
2
3       int main()
4       {
5               myprintf();
6               return 0;
7       }
(gdb) break 7      #在程序的末尾設置一個斷點
Breakpoint 1 at 0x80481bf: file test.c, line 7.
(gdb) r            #程序都快結束了都沒問題，怎麼會到最後出個問題呢？
Starting program: /mnt/hda8/Temp/c/program/test
hello, world!

Breakpoint 1, main () at test.c:7
7       }
(gdb) n        #單步執行看看，怎麼下面一條指令是0x00000001，肯定是程序退出以後出了問題
0x00000001 in ?? ()
(gdb) n        #誒，當然找不到邊了，都跑到0x00000001了
Cannot find bounds of current function
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00000001 in ?? ()
```

原來是這麼回事，估計是 `return 0` 返回之後出問題了，看看它的彙編去。

```
$ gcc -S test.c #產生彙編代碼
$ cat test.s
...
        call    myprintf
        movl    $0, %eax
        addl    $4, %esp
        popl    %ecx
        popl    %ebp
        leal    -4(%ecx), %esp
        ret
...
```

後面就這麼幾條指令，難不成 `ret` 返回有問題，不讓它 `ret` 返回，把 `return` 改成 `_exit` 直接進入內核退出。

```
$ vim test.c
$ cat test.c    #就把return語句修改成_exit了。
#include "test.h"
#include <unistd.h> /* _exit */

int main()
{
	myprintf();
	_exit(0);
}
$ gcc -g -c test.c myprintf.c
$ ld -dynamic-linker /lib/ld-linux.so.2 -o test test.o myprintf.o -L/usr/lib -lc
ld: warning: cannot find entry symbol _start; defaulting to 00000000080481d8
$ ./test    #竟然好了，再看看彙編有什麼不同
hello, world!
$ gcc -S test.c
$ cat test.s    #貌似就把ret指令替換成了_exit函數調用，直接進入內核，讓內核處理了，那為什麼ret有問題呢？
...
        call    myprintf
        subl    $12, %esp
        pushl   $0
        call    _exit
...
$ gdb ./test    #把代碼改回去（改成return 0;），再調試看看調用main函數返回時的下一條指令地址eip
...
(gdb) l
warning: Source file is more recent than executable.
1       #include "test.h"
2
3       int main()
4       {
5               myprintf();
6               return 0;
7       }
(gdb) break 5
Breakpoint 1 at 0x80481b5: file test.c, line 5.
(gdb) break 7
Breakpoint 2 at 0x80481bc: file test.c, line 7.
(gdb) r
Starting program: /mnt/hda8/Temp/c/program/test

Breakpoint 1, main () at test.c:5
5               myprintf();
(gdb) x/8x $esp
0xbf929510:     0xbf92953c      0x080481a4      0x00000000      0xb7eea84f
0xbf929520:     0xbf92953c      0xbf929534      0x00000000      0x00000001
```

發現 `0x00000001` 剛好是之前調試時看到的程序返回後的位置，即 `eip`，說明程序在初始化時，這個 `eip` 就是錯誤的。為什麼呢？因為根本沒有鏈接進初始化的代碼，而是在編譯器自己給我們，初始化了程序入口即 `00000000080481d8，也就是說，沒有人調用 `main`，`main` 不知道返回哪裡去，所以，我們直接讓 `main` 結束時進入內核調用 `_exit` 而退出則不會有問題。

通過上面的演示和解釋發現只要把return語句修改為_exit語句，程序即使不鏈接任何額外的目標代碼都可以正常運行（原因是不鏈接那些額外的文件時相當於沒有進行初始化操作，如果在程序的最後執行ret彙編指令，程序將無法獲得正確的eip，從而無法進行後續的動作）。但是為什麼會有“找不到_start符號”的警告呢？通過`readelf -s`查看crt1.o發現裡頭有這個符號，並且crt1.o引用了main這個符號，是不是意味著會從`_start`進入main呢？是不是程序入口是`_start`，而並非main呢？

<span id="toc_27212_14734_26"></span>
### C 語言程序真正的入口

先來看看剛才提到的鏈接器的默認鏈接腳本（`ld -m elf_386 --verbose`），它告訴我們程序的入口（entry）是 `_start`，而一個可執行文件必須有一個入口地址才能運行，所以這就是說明了為什麼 `ld` 一定要提示我們 “_start找不到”，找不到以後就給默認設置了一個地址。

```
$ ld --verbose  | grep ^ENTRY    #非交叉編譯，可不用-m參數；ld默認找_start入口，並不是main哦！
ENTRY(_start)
```

原來是這樣，程序的入口（entry）竟然不是 `main` 函數，而是 `_start`。那乾脆把彙編裡頭的 `main` 給改掉算了，看行不行？

先生成彙編 `test.s`：

```
$ cat test.c
#include "test.h"
#include <unistd.h>     /* _exit */

int main()
{
	myprintf();
	_exit(0);
}
$ gcc -S test.c
```

然後把彙編中的 `main` 改為 `_start`，即改程序入口為 `_start`：

```
$ sed -i -e "s#main#_start#g" test.s
$ gcc -c test.s myprintf.c
```

重新鏈接，發現果然沒問題了：

```
$ ld -dynamic-linker /lib/ld-linux.so.2 -o test test.o myprintf.o -L/usr/lib/ -lc
$ ./test
hello, world!
```

`_start` 竟然是真正的程序入口，那在有 `main` 的情況下呢？為什麼在 `_start` 之後能夠找到 `main` 呢？這個看看 alert7 大叔的[Before main 分析](http://www.xfocus.net/articles/200109/269.html)吧，這裡不再深入介紹。

總之呢，通過修改程序的 `return` 語句為 `_exit(0)` 和修改程序的入口為 `_start`，我們的代碼不鏈接 `gcc` 默認鏈接的那些額外的文件同樣可以工作得很好。並且打破了一個學習 C 語言以來的常識：`main` 函數作為程序的主函數，是程序的入口，實際上則不然。

<span id="toc_27212_14734_27"></span>
### 鏈接腳本初次接觸

再補充一點內容，在 `ld` 的鏈接腳本中，有一個特別的關鍵字 `PROVIDE`，由這個關鍵字定義的符號是 `ld` 的預定義字符，我們可以在 C 語言函數中擴展它們後直接使用。這些特別的符號可以通過下面的方法獲取，

```
$ ld --verbose | grep PROVIDE | grep -v HIDDEN
  PROVIDE (__executable_start = 0x08048000); . = 0x08048000 + SIZEOF_HEADERS;
  PROVIDE (__etext = .);
  PROVIDE (_etext = .);
  PROVIDE (etext = .);
  _edata = .; PROVIDE (edata = .);
  _end = .; PROVIDE (end = .);
```

這裡面有幾個我們比較關心的，第一個是程序的入口地址 `__executable_start`，另外三個是 `etext`，`edata`，`end`，分別對應程序的代碼段（text）、初始化數據（data）和未初始化的數據（bss）（可參考`man etext`），如何引用這些變量呢？看看這個例子。

```
/* predefinevalue.c */
#include <stdio.h>

extern int __executable_start, etext, edata, end;

int main(void)
{
	printf ("program entry: 0x%x \n", &__executable_start);
	printf ("etext address(text segment): 0x%x \n", &etext);
	printf ("edata address(initilized data): 0x%x \n", &edata);
	printf ("end address(uninitilized data): 0x%x \n", &end);

	return 0;
}
```

到這裡，程序鏈接過程的一些細節都介紹得差不多了。在[《動態符號鏈接的細節》][100]中將主要介紹 ELF 文件的動態符號鏈接過程。

<span id="toc_27212_14734_28"></span>
## 參考資料

- [Linux 彙編語言開發指南](http://www.ibm.com/developerworks/cn/linux/l-assembly/index.html)
- [PowerPC 彙編](http://www.ibm.com/developerworks/cn/linux/hardware/ppc/assembly/index.html)
- [用於 Power 體系結構的彙編語言](http://www.ibm.com/developerworks/cn/linux/l-powasm1.html)
- [Linux 中 x86 的內聯彙編](http://www.ibm.com/developerworks/cn/linux/sdk/assemble/inline/index.html)
- Linux Assembly HOWTO
- Linux Assembly Language Programming
- Guide to Assembly Language Programming in Linux
- [An beginners guide to compiling programs under Linux](http://www.luv.asn.au/overheads/compile.html)
- [gcc manual](http://gcc.gnu.org/onlinedocs/gcc-4.2.2/gcc/)
- [A Quick Tour of Compiling, Linking, Loading, and Handling Libraries on Unix](http://efrw01.frascati.enea.it/Software/Unix/IstrFTU/cern-cnl-2001-003-25-link.html)
- [Unix 目標文件初探](http://www.ibm.com/developerworks/cn/aix/library/au-unixtools.html)
- [Before main()分析](http://www.xfocus.net/articles/200109/269.html)
- [A Process Viewing Its Own /proc/<PID>/map Information](http://www.linuxforums.org/forum/linux-kernel/51790-process-viewing-its-own-proc-pid-map-information.html)
- UNIX 環境高級編程
- Linux Kernel Primer
- [Understanding ELF using readelf and objdump](http://www.linuxforums.org/misc/understanding_elf_using_readelf_and_objdump.html)
- [Study of ELF loading and relocs](http://netwinder.osuosl.org/users/p/patb/public_html/elf_relocs.html)
- ELF file format and ABI
    - [\[1\]](http://www.x86.org/ftp/manuals/tools/elf.pdf)
    - [\[2\]](http://www.muppetlabs.com/~breadbox/software/ELF.txt)
- TN05.ELF.Format.Summary.pdf
- [ELF文件格式(中文)](http://www.xfocus.net/articles/200105/174.html)
- 關於 Gcc 方面的論文，請查看歷年的會議論文集
    - [2005](http://www.gccsummit.org/2005/2005-GCC-Summit-Proceedings.pdf)
    - [2006](http://www.gccsummit.org/2006/2006-GCC-Summit-Proceedings.pdf)
- [The Linux GCC HOW TO](http://www.faqs.org/docs/Linux-HOWTO/GCC-HOWTO.html)
- [ELF: From The Programmer's Perspective](http://linux.jinr.ru/usoft/WWW/www_debian.org/Documentation/elf/elf.html)
- [C/C++ 程序編譯步驟詳解](http://www.xxlinux.com/linux/article/development/soft/20070424/8267.html)
- [C 語言常見問題集](http://c-faq-chn.sourceforge.net/ccfaq/index.html)
- [使用 BFD 操作 ELF](http://elfhack.whitecell.org/mydocs/use_bfd.txt)
- [bfd document](http://sourceware.org/binutils/docs/bfd/index.html)
- [UNIX/LINUX 平臺可執行文件格式分析](http://blog.chinaunix.net/u/19881/showart_215242.html)
- [Linux 彙編語言快速上手：4大架構一塊學](http://www.tinylab.org/linux-assembly-language-quick-start/)
- GNU binutils 小結
