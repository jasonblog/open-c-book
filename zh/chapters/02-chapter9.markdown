# 代碼測試、調試與優化

-    [前言](#toc_7140_15195_1)
-    [代碼測試](#toc_7140_15195_2)
    -    [測試程序的運行時間 time](#toc_7140_15195_3)
    -    [函數調用關係圖 calltree](#toc_7140_15195_4)
    -    [性能測試工具 gprof & kprof](#toc_7140_15195_5)
    -    [代碼覆蓋率測試 gcov & ggcov](#toc_7140_15195_6)
    -    [內存訪問越界 catchsegv, libSegFault.so](#toc_7140_15195_7)
    -    [緩衝區溢出 libsafe.so](#toc_7140_15195_8)
    -    [內存洩露 Memwatch, Valgrind, mtrace](#toc_7140_15195_9)
-    [代碼調試](#toc_7140_15195_10)
    -    [靜態調試：printf + gcc -D（打印程序中的變量）](#toc_7140_15195_11)
    -    [交互式的調試（動態調試）：gdb（支持本地和遠程）/ald（彙編指令級別的調試）](#toc_7140_15195_12)
        -    [嵌入式系統調試方法 gdbserver/gdb](#toc_7140_15195_13)
        -    [彙編代碼的調試 ald](#toc_7140_15195_14)
    -    [實時調試：gdb tracepoint](#toc_7140_15195_15)
    -    [調試內核](#toc_7140_15195_16)
-    [代碼優化](#toc_7140_15195_17)
-    [參考資料](#toc_7140_15195_18)


<span id="toc_7140_15195_1"></span>
## 前言

代碼寫完以後往往要做測試（或驗證）、調試，可能還要優化。

- 關於測試（或驗證）

  通常對應著兩個英文單詞 `Verification和 `Validation`，在資料 [\[1\]][1] 中有關於這個的定義和一些深入的討論，在資料 [\[2\]][2] 中，很多人給出了自己的看法。但是正如資料 [\[2\]][2] 提到的：

  > The differences between verification and validation are unimportant except to the theorist; practitioners use the term V&V to refer to all of the activities that are aimed at making sure the software will function as required.

  所以，無論測試（或驗證）目的都是為了讓軟件的功能能夠達到需求。測試和驗證通常會通過一些形式化（貌似可以簡單地認為有數學根據的）或者非形式化的方法去驗證程序的功能是否達到要求。

- 關於調試

  而調試對應英文 debug，debug 叫“驅除害蟲”，也許一個軟件的功能達到了要求，但是可能會在測試或者是正常運行時出現異常，因此需要處理它們。

- 關於優化

  debug 是為了保證程序的正確性，之後就需要考慮程序的執行效率，對於存儲資源受限的嵌入式系統，程序的大小也可能是優化的對象。

  很多理論性的東西實在沒有研究過，暫且不說吧。這裡只是想把一些需要動手實踐的東西先且記錄和總結一下，另外很多工具在這裡都有提到和羅列，包括 Linux 內核調試相關的方法和工具。關於更詳細更深入的內容還是建議直接看後面的參考資料為妙。

下面的所有演示在如下環境下進行：

```
$ uname -a
Linux falcon 2.6.22-14-generic #1 SMP Tue Feb 12 07:42:25 UTC 2008 i686 GNU/Linux
$ echo $SHELL
/bin/bash
$ /bin/bash --version | grep bash
GNU bash, version 3.2.25(1)-release (i486-pc-linux-gnu)
$ gcc --version | grep gcc
gcc (GCC) 4.1.3 20070929 (prerelease) (Ubuntu 4.1.2-16ubuntu2)
$ cat /proc/cpuinfo | grep "model name"
model name      : Intel(R) Pentium(R) 4 CPU 2.80GHz
```

<span id="toc_7140_15195_2"></span>
## 代碼測試

代碼測試有很多方面，例如運行時間、函數調用關係圖、代碼覆蓋度、性能分析（Profiling）、內存訪問越界（Segmentation Fault）、緩衝區溢出（Stack Smashing 合法地進行非法的內存訪問？所以很危險）、內存洩露（Memory Leak）等。

<span id="toc_7140_15195_3"></span>
### 測試程序的運行時間 time

Shell 提供了內置命令 `time` 用於測試程序的執行時間，默認顯示結果包括三部分：實際花費時間（real time）、用戶空間花費時間（user time）和內核空間花費時間（kernel time）。

```
$ time pstree 2>&1 >/dev/null

real    0m0.024s
user    0m0.008s
sys     0m0.004s
```

`time` 命令給出了程序本身的運行時間。這個測試原理非常簡單，就是在程序運行（通過 `system` 函數執行）前後記錄了系統時間（用 `times` 函數），然後進行求差就可以。如果程序運行時間很短，運行一次看不到效果，可以考慮採用測試紙片厚度的方法進行測試，類似把很多紙張疊到一起來測試紙張厚度一樣，我們可以讓程序運行很多次。

如果程序運行時間太長，執行效率很低，那麼得考慮程序內部各個部分的執行情況，從而對代碼進行可能的優化。具體可能會考慮到這兩點：

對於 C 語言程序而言，一個比較宏觀的層次性的輪廓（profile）是函數調用圖、函數內部的條件分支構成的語句塊，然後就是具體的語句。把握好這樣一個輪廓後，就可以有針對性地去關注程序的各個部分，包括哪些函數、哪些分支、哪些語句最值得關注（執行次數越多越值得優化，術語叫 hotspots）。

對於 Linux 下的程序而言，程序運行時涉及到的代碼會涵蓋兩個空間，即用戶空間和內核空間。由於這兩個空間涉及到地址空間的隔離，在測試或調試時，可能涉及到兩個空間的工具。前者絕大多數是基於 `Gcc` 的特定參數和系統的 `ptrace` 調用，而後者往往實現為內核的補丁，它們在原理上可能類似，但實際操作時後者顯然會更麻煩，不過如果你不去 hack 內核，那麼往往無須關心後者。

<span id="toc_7140_15195_4"></span>
### 函數調用關係圖 calltree

`calltree` 可以非常簡單方便地反應一個項目的函數調用關係圖，雖然諸如 `gprof` 這樣的工具也能做到，不過如果僅僅要得到函數調用圖，`calltree` 應該是更好的選擇。如果要產生圖形化的輸出可以使用它的 `-dot` 參數。從[這裡](ftp://ftp.berlios.de/pub/calltree/calltree-2.3.tar.bz2)可以下載到它。

這裡是一份基本用法演示結果：

```
$ calltree -b -np -m *.c
main:
|   close
|   commitchanges
|   |   err
|   |   |   fprintf
|   |   ferr
|   |   ftruncate
|   |   lseek
|   |   write
|   ferr
|   getmemorysize
|   modifyheaders
|   open
|   printf
|   readelfheader
|   |   err
|   |   |   fprintf
|   |   ferr
|   |   read
|   readphdrtable
|   |   err
|   |   |   fprintf
|   |   ferr
|   |   malloc
|   |   read
|   truncatezeros
|   |   err
|   |   |   fprintf
|   |   ferr
|   |   lseek
|   |   read$ 
```

這樣一份結果對於“反向工程”應該會很有幫助，它能夠呈現一個程序的大體結構，對於閱讀和分析源代碼來說是一個非常好的選擇。雖然 `cscope` 和 `ctags` 也能夠提供一個函數調用的“即時”（在編輯 Vim 的過程中進行調用）視圖（view），但是 `calltree` 卻給了我們一個宏觀的視圖。

不過這樣一個視圖只涉及到用戶空間的函數，如果想進一步給出內核空間的宏觀視圖，那麼 `strace`，`KFT` 或者 `Ftrace` 就可以發揮它們的作用。另外，該視圖也沒有給出庫中的函數，如果要跟蹤呢？需要 `ltrace` 工具。

另外發現 `calltree` 僅僅給出了一個程序的函數調用視圖，而沒有告訴我們各個函數的執行次數等情況。如果要關注這些呢？我們有 `gprof`。

<span id="toc_7140_15195_5"></span>
### 性能測試工具 gprof & kprof

參考資料[\[3\]][3]詳細介紹了這個工具的用法，這裡僅挑選其中一個例子來演示。`gprof` 是一個命令行的工具，而 KDE 桌面環境下的 `kprof` 則給出了圖形化的輸出，這裡僅演示前者。

首先來看一段代碼（來自資料[\[3\]][3]），算 `Fibonacci` 數列的，

```
#include <stdio.h>

int fibonacci(int n);

int main (int argc, char **argv)
{
	int fib;
	int n;

	for (n = 0; n <= 42; n++) {
		fib = fibonacci(n);
		printf("fibonnaci(%d) = %d\n", n, fib);
	}

	return 0;
}

int fibonacci(int n)
{
	int fib;

	if (n <= 0) {
		fib = 0;
	} else if (n == 1) {
		fib = 1;
	} else {
		fib = fibonacci(n -1) + fibonacci(n - 2);
	}

	return fib;
}
```

通過 `calltree` 看看這段代碼的視圖，

```
$ calltree -b -np -m *.c
main:
|   fibonacci
|   |   fibonacci ....
|   printf
```

可以看出程序主要涉及到一個 `fibonacci` 函數，這個函數遞歸調用自己。為了能夠使用 `gprof`，需要編譯時加上 `-pg` 選項，讓 `Gcc` 加入相應的調試信息以便 `gprof` 能夠產生函數執行情況的報告。

```
$ gcc -pg -o fib fib.c
$ ls
fib  fib.c
```

運行程序並查看執行時間，

```
$ time ./fib
fibonnaci(0) = 0
fibonnaci(1) = 1
fibonnaci(2) = 1
fibonnaci(3) = 2
...
fibonnaci(41) = 165580141
fibonnaci(42) = 267914296

real    1m25.746s
user    1m9.952s
sys     0m0.072s
$ ls
fib  fib.c  gmon.out
```

上面僅僅選取了部分執行結果，程序運行了 1 分多鐘，代碼運行以後產生了一個 `gmon.out` 文件，這個文件可以用於 `gprof` 產生一個相關的性能報告。

```
$ gprof  -b ./fib gmon.out 
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 96.04     14.31    14.31       43   332.80   332.80  fibonacci
  4.59     14.99     0.68                             main


                        Call graph


granularity: each sample hit covers 2 byte(s) for 0.07% of 14.99 seconds

index % time    self  children    called     name
                                                 <spontaneous>
[1]    100.0    0.68   14.31                 main [1]
               14.31    0.00      43/43          fibonacci [2]
-----------------------------------------------
                             2269806252             fibonacci [2]
               14.31    0.00      43/43          main [1]
[2]     95.4   14.31    0.00      43+2269806252 fibonacci [2]
                             2269806252             fibonacci [2]
-----------------------------------------------


Index by function name

   [2] fibonacci               [1] main
```

從這份結果中可觀察到程序中每個函數的執行次數等情況，從而找出值得修改的函數。在對某些部分修改之後，可以再次比較程序運行時間，查看優化結果。另外，這份結果還包含一個特別有用的東西，那就是程序的動態函數調用情況，即程序運行過程中實際執行過的函數，這和 `calltree` 產生的靜態調用樹有所不同，它能夠反應程序在該次執行過程中的函數調用情況。而如果想反應程序運行的某一時刻調用過的函數，可以考慮採用 `gdb` 的 `backtrace` 命令。

類似測試紙片厚度的方法，`gprof` 也提供了一個統計選項，用於對程序的多次運行結果進行統計。另外，`gprof` 有一個 KDE 下圖形化接口 `kprof`，這兩部分請參考資料[\[3\]][3]。

對於非 KDE 環境，可以使用 [Gprof2Dot](https://code.google.com/p/jrfonseca/wiki/Gprof2Dot) 把 `gprof` 輸出轉換成圖形化結果。

關於 `dot` 格式的輸出，也可以可以考慮通過 `dot` 命令把結果轉成 `jpg` 等格式，例如：

```
$ dot -Tjpg test.dot -o test.jp
```

`gprof` 雖然給出了函數級別的執行情況，但是如果想關心具體哪些條件分支被執行到，哪些語句沒有被執行，該怎麼辦？

<span id="toc_7140_15195_6"></span>
### 代碼覆蓋率測試 gcov & ggcov

如果要使用 `gcov`，在編譯時需要加上這兩個選項 `-fprofile-arcs -ftest-coverage`，這裡直接用之前的 `fib.c` 做演示。

```
$ ls
fib.c
$ gcc -fprofile-arcs -ftest-coverage -o fib fib.c
$ ls
fib  fib.c  fib.gcno
```
運行程序，並通過 `gcov` 分析代碼的覆蓋度：

```
$ ./fib
$ gcov fib.c
File 'fib.c'
Lines executed:100.00% of 12
fib.c:creating 'fib.c.gcov'
```

12 行代碼 100% 被執行到，再查看分支情況，

```
$ gcov -b fib.c
File 'fib.c'
Lines executed:100.00% of 12
Branches executed:100.00% of 6
Taken at least once:100.00% of 6
Calls executed:100.00% of 4
fib.c:creating 'fib.c.gcov'
```

發現所有函數，條件分支和語句都被執行到，說明代碼的覆蓋率很高，不過資料[\[3\]][3] `gprof` 的演示顯示代碼的覆蓋率高並不一定說明代碼的性能就好，因為那些被覆蓋到的代碼可能能夠被優化成性能更高的代碼。那到底哪些代碼值得被優化呢？執行次數最多的，另外，有些分支雖然都覆蓋到了，但是這個分支的位置可能並不是理想的，如果一個分支的內容被執行的次數很多，那麼把它作為最後一個分支的話就會浪費很多不必要的比較時間。因此，通過覆蓋率測試，可以嘗試著剔除那些從未執行過的代碼或者把那些執行次數較多的分支移動到較早的條件分支裡頭。通過性能測試，可以找出那些值得優化的函數、分支或者是語句。

如果使用 `-fprofile-arcs -ftest-coverage` 參數編譯完代碼，可以接著用 `-fbranch-probabilities` 參數對代碼進行編譯，這樣，編譯器就可以對根據代碼的分支測試情況進行優化。

```
$ wc -c fib
16333 fib
$ ls fib.gcda  #確保fib.gcda已經生成，這個是運行fib後的結果
fib.gcda
$ gcc -fbranch-probabilities -o fib fib.c #再次運行
$ wc -c fib
6604 fib
$ time ./fib
...
real    0m21.686s
user    0m18.477s
sys     0m0.008s
```

可見代碼量減少了，而且執行效率會有所提高，當然，這個代碼效率的提高可能還跟其他因素有關，比如 `Gcc` 還優化了一些跟平臺相關的指令。

如果想看看代碼中各行被執行的情況，可以直接看 `fib.c.gcov` 文件。這個文件的各列依次表示執行次數、行號和該行的源代碼。次數有三種情況，如果一直沒有執行，那麼用 `####` 表示；如果該行是註釋、函數聲明等，用 `-` 表示；如果是純粹的代碼行，那麼用執行次數表示。這樣我們就可以直接分析每一行的執行情況。

`gcov` 也有一個圖形化接口 `ggcov`，是基於 `gtk+` 的，適合 Gnome 桌面的用戶。

現在都已經關注到代碼行了，實際上優化代碼的前提是保證代碼的正確性，如果代碼還有很多 bug，那麼先要 debug。不過下面的這些 "bug" 用普通的工具確實不太方便，雖然可能，不過這裡還是把它們歸結為測試的內容，並且這裡剛好承接上 `gcov` 部分，`gcov` 能夠測試到每一行的代碼覆蓋情況，而無論是內存訪問越界、緩衝區溢出還是內存洩露，實際上是發生在具體的代碼行上的。

<span id="toc_7140_15195_7"></span>
### 內存訪問越界 catchsegv, libSegFault.so

"Segmentation fault" 是很頭痛的一個問題，估計“糾纏”過很多人。這裡僅僅演示通過 `catchsegv` 腳本測試段錯誤的方法，其他方法見後面相關資料。

`catchsegv` 利用系統動態鏈接的 `PRELOAD` 機制（請參考`man ld-linux`），把庫 `/lib/libSegFault.so` 提前 load 到內存中，然後通過它檢查程序運行過程中的段錯誤。

```
$ cat test.c
#include <stdio.h>

int main(void)
{
	char str[10];

        sprintf(str, "%s", 111);

        printf("str = %s\n", str);
        return 0;
}
$ make test
$ LD_PRELOAD=/lib/libSegFault.so ./test  #等同於catchsegv ./test
*** Segmentation fault
Register dump:

 EAX: 0000006f   EBX: b7eecff4   ECX: 00000003   EDX: 0000006f
 ESI: 0000006f   EDI: 0804851c   EBP: bff9a8a4   ESP: bff9a27c

 EIP: b7e1755b   EFLAGS: 00010206

 CS: 0073   DS: 007b   ES: 007b   FS: 0000   GS: 0033   SS: 007b

 Trap: 0000000e   Error: 00000004   OldMask: 00000000
 ESP/signal: bff9a27c   CR2: 0000006f

Backtrace:
/lib/libSegFault.so[0xb7f0604f]
[0xffffe420]
/lib/tls/i686/cmov/libc.so.6(vsprintf+0x8c)[0xb7e0233c]
/lib/tls/i686/cmov/libc.so.6(sprintf+0x2e)[0xb7ded9be]
./test[0x804842b]
/lib/tls/i686/cmov/libc.so.6(__libc_start_main+0xe0)[0xb7dbd050]
./test[0x8048391]
...
```

從結果中可以看出，代碼的 `sprintf` 有問題。經過檢查發現它把整數當字符串輸出，對於字符串的輸出，需要字符串的地址作為參數，而這裡的 `111` 則剛好被解釋成了字符串的地址，因此 `sprintf` 試圖訪問 `111` 這個地址，從而發生了非法訪問內存的情況，出現 “Segmentation Fault”。

<span id="toc_7140_15195_8"></span>
### 緩衝區溢出 libsafe.so

緩衝區溢出是堆棧溢出（Stack Smashing），通常發生在對函數內的局部變量進行賦值操作時，超出了該變量的字節長度而引起對棧內原有數據（比如 eip，ebp 等）的覆蓋，從而引發內存訪問越界，甚至執行非法代碼，導致系統崩潰。關於緩衝區的詳細原理和實例分析見[《緩衝區溢出與注入分析》][100]。這裡僅僅演示該資料中提到的一種用於檢查緩衝區溢出的方法，它同樣採用動態鏈接的 `PRELOAD` 機制提前裝載一個名叫 `libsafe.so` 的庫，可以從[這裡](http://www.sfr-fresh.com/linux/misc/libsafe-2.0-16.tgz)獲取它，下載後，再解壓，編譯，得到 `libsafe.so`，

[100]: 02-chapter5.markdown

下面，演示一個非常簡單的，但可能存在緩衝區溢出的代碼，並演示 `libsafe.so` 的用法。

```
$ cat test.c
$ make test
$ LD_PRELOAD=/path/to/libsafe.so ./test ABCDEFGHIJKLMN
ABCDEFGHIJKLMN
*** stack smashing detected ***: ./test terminated
Aborted (core dumped)
```

資料[\[7\]][7]分析到，如果不能夠對緩衝區溢出進行有效的處理，可能會存在很多潛在的危險。雖然 `libsafe.so` 採用函數替換的方法能夠進行對這類 Stack Smashing 進行一定的保護，但是無法根本解決問題，alert7 大蝦在資料[\[10\]][10]中提出了突破它的辦法，資料[11]11]提出了另外一種保護機制。

<span id="toc_7140_15195_9"></span>
### 內存洩露 Memwatch, Valgrind, mtrace

堆棧通常會被弄在一起叫，不過這兩個名詞卻是指進程的內存映像中的兩個不同的部分，棧（Stack）用於函數的參數傳遞、局部變量的存儲等，是系統自動分配和回收的；而堆（heap）則是用戶通過 `malloc` 等方式申請而且需要用戶自己通過 `free` 釋放的，如果申請的內存沒有釋放，那麼將導致內存洩露，進而可能導致堆的空間被用盡；而如果已經釋放的內存再次被釋放（double-free）則也會出現非法操作。如果要真正理解堆和棧的區別，需要理解進程的內存映像，請參考[《緩衝區溢出與注入分析》][100]

這裡演示通過 `Memwatch` 來檢測程序中可能存在內存洩露，可以從[這裡](http://www.linkdata.se/sourcecode.html)下載到這個工具。
使用這個工具的方式很簡單，只要把它鏈接（ld）到可執行文件中去，並在編譯時加上兩個宏開關`-DMEMWATCH -DMW_STDIO`。這裡演示一個簡單的例子。

```
$ cat test.c 
#include <stdlib.h>
#include <stdio.h>
#include "memwatch.h"

int main(void)
{
	char *ptr1;
	char *ptr2;

	ptr1 = malloc(512);
	ptr2 = malloc(512);

	ptr2 = ptr1;
	free(ptr2);
	free(ptr1);
}
$ gcc -DMEMWATCH -DMW_STDIO test.c memwatch.c -o test
$ cat memwatch.log
============= MEMWATCH 2.71 Copyright (C) 1992-1999 Johan Lindh =============

Started at Sat Mar  1 07:34:33 2008

Modes: __STDC__ 32-bit mwDWORD==(unsigned long)
mwROUNDALLOC==4 sizeof(mwData)==32 mwDataSize==32

double-free: <4> test.c(15), 0x80517e4 was freed from test.c(14)

Stopped at Sat Mar  1 07:34:33 2008

unfreed: <2> test.c(11), 512 bytes at 0x8051a14         {FE FE FE FE FE FE FE FE FE FE FE FE FE FE FE FE ................}

Memory usage statistics (global):
 N)umber of allocations made: 2
 L)argest memory usage      : 1024
 T)otal of all alloc() calls: 1024
 U)nfreed bytes totals      : 512
```

通過測試，可以看到有一個 512 字節的空間沒有被釋放，而另外 512 字節空間卻被連續釋放兩次（double-free）。`Valgrind` 和 `mtrace` 也可以做類似的工作，請參考資料[\[4\]][4]，[\[5\]][5]和 `mtrace` 的手冊。

<span id="toc_7140_15195_10"></span>
## 代碼調試

調試的方法很多，調試往往要跟蹤代碼的運行狀態，`printf` 是最基本的辦法，然後呢？靜態調試方法有哪些，非交互的呢？非實時的有哪些？實時的呢？用於調試內核的方法有哪些？有哪些可以用來調試彙編代碼呢？

<span id="toc_7140_15195_11"></span>
### 靜態調試：printf + gcc -D（打印程序中的變量）

利用 `Gcc` 的宏定義開關（`-D`）和 `printf` 函數可以跟蹤程序中某個位置的狀態，這個狀態包括當前一些變量和寄存器的值。調試時需要用 `-D` 開關進行編譯，在正式發佈程序時則可把 `-D` 開關去掉。這樣做比單純用 `printf` 方便很多，它可以避免清理調試代碼以及由此帶來的代碼誤刪除等問題。

```
$ cat test.c
#include <stdio.h>
#include <unistd.h>

int main(void)
{
	int i = 0;

#ifdef DEBUG
        printf("i = %d\n", i);

        int t;
        __asm__ __volatile__ ("movl %%ebp, %0;":"=r"(t)::"%ebp");
        printf("ebp = 0x%x\n", t);
#endif

        _exit(0);
}
$ gcc -DDEBUG -g -o test test.c
$ ./test
i = 0
ebp = 0xbfb56d98
```

上面演示瞭如何跟蹤普通變量和寄存器變量的辦法。跟蹤寄存器變量採用了內聯彙編。

不過，這種方式不夠靈活，我們無法“即時”獲取程序的執行狀態，而 `gdb` 等交互式調試工具不僅解決了這樣的問題，而且通過把調試器拆分成調試服務器和調試客戶端適應了嵌入式系統的調試，另外，通過預先設置斷點以及斷點處需要收集的程序狀態信息解決了交互式調試不適應實時調試的問題。

<span id="toc_7140_15195_12"></span>
### 交互式的調試（動態調試）：gdb（支持本地和遠程）/ald（彙編指令級別的調試）

<span id="toc_7140_15195_13"></span>
#### 嵌入式系統調試方法 gdbserver/gdb

估計大家已經非常熟悉 GDB（Gnu DeBugger）了，所以這裡並不介紹常規的 `gdb` 用法，而是介紹它的服務器／客戶（`gdbserver/gdb`）調試方式。這種方式非常適合嵌入式系統的調試，為什麼呢？先來看看這個：

```
$ wc -c /usr/bin/gdbserver 
56000 /usr/bin/gdbserver
$ which gdb
/usr/bin/gdb
$ wc -c /usr/bin/gdb
2557324 /usr/bin/gdb
$ echo "(2557324-56000)/2557324"  | bc -l
.97810210986171482377
```

`gdb` 比 `gdbserver` 大了將近 97%，如果把整個 `gdb` 搬到存儲空間受限的嵌入式系統中是很不合適的，不過僅僅 5K 左右的 `gdbserver` 即使在只有 8M Flash 卡的嵌入式系統中也都足夠了。所以在嵌入式開發中，我們通常先在本地主機上交叉編譯好 `gdbserver/gdb`。

如果是初次使用這種方法，可能會遇到麻煩，而麻煩通常發生在交叉編譯 `gdb` 和 `gdbserver` 時。在編譯 `gdbserver/gdb` 前，需要配置(./configure)兩個重要的選項：

- `--host`，指定 gdb/gdbserver 本身的運行平臺，
- `--target`，指定 gdb/gdbserver 調試的代碼所運行的平臺，

關於運行平臺，通過 `$MACHTYPE` 環境變量就可獲得，對於 `gdbserver`，因為要把它複製到嵌入式目標系統上，並且用它來調試目標平臺上的代碼，因此需要把 `--host` 和 `--target` 都設置成目標平臺；而 `gdb` 因為還是運行在本地主機上，但是需要用它調試目標系統上的代碼，所以需要把 `--target` 設置成目標平臺。

編譯完以後就是調試，調試時需要把程序交叉編譯好，並把二進制文件複製一份到目標系統上，並在本地需要保留一份源代碼文件。調試過程大體如下，首先在目標系統上啟動調試服務器：

```
$ gdbserver :port /path/to/binary_file
...
```

然後在本地主機上啟動gdb客戶端鏈接到 `gdb` 調試服務器，（`gdbserver_ipaddress` 是目標系統的IP地址，如果目標系統不支持網絡，那麼可以採用串口的方式，具體看手冊）

```
$ gdb
...
(gdb) target remote gdbserver_ipaddress:2345
...
```

其他調試過程和普通的gdb調試過程類似。

<span id="toc_7140_15195_14"></span>
#### 彙編代碼的調試 ald

用 `gdb` 調試彙編代碼貌似會比較麻煩，不過有人正是因為這個原因而開發了一個專門的彙編代碼調試器，名字就叫做 `assembly language debugger`，簡稱 `ald`，你可以從[這裡](http://ald.sourceforge.net/)下載到。

下載後，解壓編譯，我們來調試一個程序看看。

這裡是一段非常簡短的彙編代碼：

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

彙編、鏈接、運行：

```
$ as -o test.o test.s
$ ld -o test test.o
$ ./test "Hello World"
Hello World
```

查看程序的入口地址：

```
$ readelf -h test | grep Entry 
  Entry point address:               0x8048054
```

接著用 `ald` 調試： 

```
$ ald test
ald> display
Address 0x8048054 added to step display list
ald> n
eax = 0x00000000 ebx = 0x00000000 ecx = 0x00000001 edx = 0x00000000 
esp = 0xBFBFDEB4 ebp = 0x00000000 esi = 0x00000000 edi = 0x00000000 
ds  = 0x007B es  = 0x007B fs  = 0x0000 gs  = 0x0000 
ss  = 0x007B cs  = 0x0073 eip = 0x08048055 eflags = 0x00200292 

Flags: AF SF IF ID 

Dumping 64 bytes of memory starting at 0x08048054 in hex
08048054:  59 59 59 C6 41 0C 0A 31 D2 B2 0D 31 C0 B0 04 31    YYY.A..1...1...1
08048064:  DB CD 80 31 C0 40 CD 80 00 2E 73 79 6D 74 61 62    ...1.@....symtab
08048074:  00 2E 73 74 72 74 61 62 00 2E 73 68 73 74 72 74    ..strtab..shstrt
08048084:  61 62 00 2E 74 65 78 74 00 00 00 00 00 00 00 00    ab..text........

08048055                      59                   pop ecx
```

可見 `ald` 在啟動時就已經運行了被它調試的 `test` 程序，並且進入了程序的入口 `0x8048054`，緊接著單步執行時，就執行了程序的第一條指令 `popl ecx`。

`ald` 的命令很少，而且跟 `gdb` 很類似，比如這個幾個命令用法和名字都類似 `help,next,continue,set args,break,file,quit,disassemble,enable,disable` 等。名字不太一樣但功能對等的有：`examine` 對 `x`, `enter` 對 `set variable {int} 地址=數據`。

需要提到的是：Linux 下的調試器包括上面的 `gdb` 和 `ald`，以及 `strace` 等都用到了 Linux 系統提供的 ptrace() 系統調用，這個調用為用戶訪問內存映像提供了便利，如果想自己寫一個調試器或者想hack一下 `gdb` 和 `ald`，那麼好好閱讀資料[12]和 `man ptrace` 吧。

如果確實需要用gdb調試彙編，可以參考：

- [Linux Assembly "Hello World" Tutorial, CS 200](http://web.cecs.pdx.edu/~bjorn/CS200/linux_tutorial/)
- [Debugging your Alpha Assembly Programs using GDB](http://lab46.corning-cc.edu/Documentation-Assembly_GDB_Debugger.php)

<span id="toc_7140_15195_15"></span>
### 實時調試：gdb tracepoint

對於程序狀態受時間影響的程序，用上述普通的設置斷點的交互式調試方法並不合適，因為這種方式將由於交互時產生的通信延遲和用戶輸入命令的時延而完全改變程序的行為。所以 `gdb` 提出了一種方法以便預先設置斷點以及在斷點處需要獲取的程序狀態，從而讓調試器自動執行斷點處的動作，獲取程序的狀態，從而避免在斷點處出現人機交互產生時延改變程序的行為。

這種方法叫 `tracepoints`（對應 `breakpoint`），它在 `gdb` 的用戶手冊裡頭有詳細的說明，見 [Tracepoints](https://sourceware.org/gdb/onlinedocs/gdb/Tracepoints.html)。

在內核中，有實現了相應的支持，叫 [KGTP](http://lwn.net/Articles/538818/)。

<span id="toc_7140_15195_16"></span>
### 調試內核

雖然這裡並不會演示如何去 hack 內核，但是相關的工具還是需要簡單提到的，[這個資料](http://elinux.org/images/c/c6/Tools_slides.pdf)列出了絕大部分用於內核調試的工具，這些對你 hack 內核應該會有幫助的。

<span id="toc_7140_15195_17"></span>
## 代碼優化

這部分暫時沒有準備足夠的素材，有待進一步完善。

暫且先提到兩個比較重要的工具，一個是 Oprofile，另外一個是 Perf。

實際上呢？“代碼測試”部分介紹的很多工具是為代碼優化服務的，更多具體的細節請參考後續資料，自己做實驗吧。

<span id="toc_7140_15195_18"></span>
## 參考資料

- [VERIFICATION AND VALIDATION][1]
- [difference between verification and Validation][2]
- [Coverage Measurement and Profiling(覆蓋度測量和性能測試,Gcov and Gprof)][3]
- Valgrind Usage
    - [Valgrind HOWTO][4]
    - [Using Valgrind to Find Memory Leaks and Invalid Memory Use][5]
- [MEMWATCH][6]
- [Mastering Linux debugging techniques][7]
- [Software Performance Analysis][8]
- [Runtime debugging in embedded systems][9]
- [繞過libsafe的保護--覆蓋_dl_lookup_versioned_symbol技術][10]
- [介紹Propolice怎樣保護stack-smashing的攻擊][11]
- Tools Provided by System：ltrace,mtrace,strace
- [Process Tracing Using Ptrace][12]
- Kernel Debugging Related Tools：KGDB, KGOV, KFI/KFT/Ftrace, GDB Tracepoint，UML, kdb
- 用Graphviz 可視化函數調用
- [Linux 段錯誤詳解](http://www.tinylab.org/explor-linux-segmentation-fault/)
- 源碼分析之函數調用關係繪製系列
    - [源碼分析：靜態分析 C 程序函數調用關係圖][50]
    - [源碼分析：動態分析 C 程序函數調用關係][51]
    - [源碼分析：動態分析 Linux 內核函數調用關係][52]
    - [源碼分析：函數調用關係繪製方法與逆向建模][53]
- Linux 下緩衝區溢出攻擊的原理及對策
- Linux 彙編語言開發指南
- [緩衝區溢出與注入分析(第一部分：進程的內存映像)][100]
- Optimizing C Code
- Performance programming for scientific computing
- Performance Programming
- Linux Profiling and Optimization
- High-level code optimization
- Code Optimization

 [1]: http://satc.gsfc.nasa.gov/assure/agbsec5.txt
 [2]: http://www.faqs.org/qa/qa-9060.html
 [3]: http://www.linuxjournal.com/article/6758
 [4]: http://www.faqs.org/docs/Linux-HOWTO/Valgrind-HOWTO.html
 [5]: http://www.cprogramming.com/debugging/valgrind.html
 [6]: http://www.linkdata.se/sourcecode.html
 [7]: http://www.ibm.com/developerworks/linux/library/l-debug/
 [8]: http://arxiv.org/pdf/cs.PF/0507073.pdf
 [9]: http://dslab.lzu.edu.cn/docs/publications/runtime_debug.pdf
 [10]: http://www.xfocus.net/articles/200208/423.html
 [11]: http://www.xfocus.net/articles/200103/78.html
 [12]: http://linuxgazette.net/issue81/sandeep.html

 [50]: http://www.tinylab.org/callgraph-draw-the-calltree-of-c-functions/
 [51]: http://www.tinylab.org/source-code-analysis-gprof2dot-draw-a-runtime-function-calls-the-c-program/
 [52]: http://www.tinylab.org/source-code-analysis-dynamic-analysis-of-linux-kernel-function-calls/
 [53]: http://www.tinylab.org/source-code-analysis-how-best-to-draw-a-function-call/
