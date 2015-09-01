# 動態符號鏈接的細節

-    [前言](#toc_23258_14315_1)
-    [基本概念](#toc_23258_14315_2)
    -    [ELF](#toc_23258_14315_3)
    -    [符號](#toc_23258_14315_4)
    -    [重定位：[是將符號引用與符號定義進行鏈接的過程][8]](#toc_23258_14315_5)
    -    [動態鏈接](#toc_23258_14315_6)
    -    [動態鏈接庫](#toc_23258_14315_7)
    -    [動態鏈接器（dynamic linker/loader）](#toc_23258_14315_8)
    -    [過程鏈接表（plt）](#toc_23258_14315_9)
    -    [全局偏移表（got）](#toc_23258_14315_10)
    -    [重定位表](#toc_23258_14315_11)
-    [動態鏈接庫的創建和調用](#toc_23258_14315_12)
    -    [創建動態鏈接庫](#toc_23258_14315_13)
    -    [隱式使用該庫](#toc_23258_14315_14)
    -    [顯式使用庫](#toc_23258_14315_15)
-    [動態鏈接過程](#toc_23258_14315_16)
-    [參考資料](#toc_23258_14315_17)


<span id="toc_23258_14315_1"></span>
## 前言

Linux 支持動態鏈接庫，不僅節省了磁盤、內存空間，而且[可以提高程序運行效率][1]。不過引入動態鏈接庫也可能會帶來很多問題，例如[動態鏈接庫的調試][4]、[升級更新][5]和潛在的安全威脅[\[1\]][6], [\[2\]][7]。這裡主要討論符號的動態鏈接過程，即程序在執行過程中，對其中包含的一些未確定地址的符號進行重定位的過程[\[1\]][3], [\[2\]][8]。

本篇主要參考資料[\[3\]][3]和[\[8\]][8]，前者側重實踐，後者側重原理，把兩者結合起來就方便理解程序的動態鏈接過程了。另外，動態鏈接庫的創建、使用以及調用動態鏈接庫的部分參考了資料[\[1\]][1], [\[2\]][2]。

下面先來看看幾個基本概念，接著就介紹動態鏈接庫的創建、隱式和顯示調用，最後介紹符號的動態鏈接細節。

<span id="toc_23258_14315_2"></span>
## 基本概念

<span id="toc_23258_14315_3"></span>
### ELF

`ELF` 是 Linux 支持的一種程序文件格式，本身包含重定位、執行、共享（動態鏈接庫）三種類型（`man elf`）。

代碼：

```
/* test.c */
#include <stdio.h>    

int global = 0;

int main()
{
        char local = 'A';

        printf("local = %c, global = %d\n", local, global);

        return 0;
}
```

演示：

通過 `-c` 生成可重定位文件 `test.o`，這裡不會進行鏈接：

```
$ gcc -c test.c
$ file test.o
test.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
```

鏈接後才可以執行：

```
$ gcc -o test test.o
$ file test
test: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
```

也可鏈接成動態鏈接庫，不過一般不會把 `main` 函數鏈接成動態鏈接庫，後面再介紹：

```
$ gcc -fpic -shared -W1,-soname,libtest.so.0 -o libtest.so.0.0 test.o
$ file libtest.so.0.0
libtest.so.0.0: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), not stripped
```

雖然 `ELF` 文件本身就支持三種不同的類型，不過它有一個統一的結構。這個結構是：

```
文件頭部(ELF Header)
程序頭部表(Program Header Table)
節區1(Section1)
節區2(Section2)
節區3(Section3)
...
節區頭部表(Section Header Table)
```

無論是文件頭部、程序頭部表、節區頭部表，還是節區，它們都對應著 C 語言裡頭的一些結構體（`elf.h` 中定義）。文件頭部主要描述 `ELF` 文件的類型，大小，運行平臺，以及和程序頭部表和節區頭部表相關的信息。節區頭部表則用於可重定位文件，以便描述各個節區的信息，這些信息包括節區的名字、類型、大小等。程序頭部表則用於描述可執行文件或者動態鏈接庫，以便系統加載和執行它們。而節區主要存放各種特定類型的信息，比如程序的正文區（代碼）、數據區（初始化和未初始化的數據）、調試信息、以及用於動態鏈接的一些節區，比如解釋器（`.interp`）節區將指定程序動態裝載 `/` 鏈接器 `ld-linux.so` 的位置，而過程鏈接表（`plt`）、全局偏移表（`got`）、重定位表則用於輔助動態鏈接過程。

<span id="toc_23258_14315_4"></span>
### 符號

對於可執行文件除了編譯器引入的一些符號外，主要就是用戶自定義的全局變量，函數等，而對於可重定位文件僅僅包含用戶自定義的一些符號。

- 生成可重定位文件

        $ gcc -c test.c
        $ nm test.o
        00000000 B global
        00000000 T main
                 U printf

    上面包含全局變量、自定義函數以及動態鏈接庫中的函數，但不包含局部變量，而且發現這三個符號的地址都沒有確定。

    注： `nm` 命令可用來查看 `ELF` 文件的符號表信息。

- 生成可執行文件


        $ gcc -o test test.o
        $ nm test | egrep "main$| printf|global$"
        080495a0 B global
        08048354 T main
                 U printf@@GLIBC_2.0

    經鏈接，`global` 和 `main` 的地址都已經確定了，但是 `printf` 卻還沒，因為它是動態鏈接庫 `glibc` 中定義函數，需要動態鏈接，而不是這裡的“靜態”鏈接。

<span id="toc_23258_14315_5"></span>
### 重定位：[是將符號引用與符號定義進行鏈接的過程][8]

從上面的演示可以看出，重定位文件 `test.o` 中的符號地址都是沒有確定的，而經過靜態鏈接（`gcc` 默認調用 `ld` 進行鏈接）以後有兩個符號地址已經確定了，這樣一個確定符號地址的過程實際上就是鏈接的實質。鏈接過後，對符號的引用變成了對地址（定義符號時確定該地址）的引用，這樣程序運行時就可通過訪問內存地址而訪問特定的數據。

我們也注意到符號 `printf` 在可重定位文件和可執行文件中的地址都沒有確定，這意味著該符號是一個外部符號，可能定義在動態鏈接庫中，在程序運行時需要通過動態鏈接器（`ld-linux.so`）進行重定位，即動態鏈接。

通過這個演示可以看出 `printf` 確實在 `glibc` 中有定義。

```
$ nm -D /lib/`uname -m`-linux-gnu/libc.so.6 | grep "\ printf$"
0000000000053840 T printf
```

除了 `nm` 以外，還可以用 `readelf -s` 查看 `.dynsym` 表或者用 `objdump -tT` 查看。

需要提到的是，用 `nm` 命令不帶 `-D` 參數的話，在較新的系統上已經沒有辦法查看 `libc.so` 的符號表了，因為 `nm` 默認打印常規符號表（在 `.symtab` 和 `.strtab` 節區中），但是，在打包時為了減少系統大小，這些符號已經被 `strip` 掉了，只保留了動態符號（在 `.dynsym` 和 `.dynstr` 中）以便動態鏈接器在執行程序時尋址這些外部用到的符號。而常規符號除了動態符號以外，還包含有一些靜態符號，比如說本地函數，這個信息主要是調試器會用，對於正常部署的系統，一般會用 `strip` 工具刪除掉。

關於 `nm` 與 `readelf -s` 的詳細比較，可參考：[nm vs “readelf -s”](http://stackoverflow.com/questions/9961473/nm-vs-readelf-s)。

<span id="toc_23258_14315_6"></span>
### 動態鏈接

動態鏈接就是在程序運行時對符號進行重定位，確定符號對應的內存地址的過程。

Linux 下符號的動態鏈接默認採用[Lazy Mode方式][3]，也就是說在程序運行過程中用到該符號時才去解析它的地址。這樣一種符號解析方式有一個好處：只解析那些用到的符號，而對那些不用的符號則永遠不用解析，從而提高程序的執行效率。

不過這種默認是可以通過設置 `LD_BIND_NOW` 為非空來打破的（下面會通過實例來分析這個變量的作用），也就是說如果設置了這個變量，動態鏈接器將在程序加載後和符號被使用之前就對這些符號的地址進行解析。

<span id="toc_23258_14315_7"></span>
### 動態鏈接庫

上面提到重定位的過程就是對符號引用和符號地址進行鏈接的過程，而動態鏈接過程涉及到的符號引用和符號定義分別對應可執行文件和動態鏈接庫，在可執行文件中可能引用了某些動態鏈接庫中定義的符號，這類符號通常是函數。

為了讓動態鏈接器能夠進行符號的重定位，必須把動態鏈接庫的相關信息寫入到可執行文件當中，這些信息是什麼呢？

```
$ readelf -d test | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
```

`ELF` 文件有一個特別的節區： `.dynamic`，它存放了和動態鏈接相關的很多信息，例如動態鏈接器通過它找到該文件使用的動態鏈接庫。不過，該信息並未包含動態鏈接庫 `libc.so.6` 的絕對路徑，那動態鏈接器去哪裡查找相應的庫呢？

通過 `LD_LIBRARY_PATH` 參數，它類似 Shell 解釋器中用於查找可執行文件的 `PATH` 環境變量，也是通過冒號分開指定了各個存放庫函數的路徑。該變量實際上也可以通過 `/etc/ld.so.conf` 文件來指定，一行對應一個路徑名。為了提高查找和加載動態鏈接庫的效率，系統啟動後會通過 `ldconfig` 工具創建一個庫的緩存 `/etc/ld.so.cache` 。如果用戶通過 `/etc/ld.so.conf` 加入了新的庫搜索路徑或者是把新庫加到某個原有的庫目錄下，最好是執行一下 `ldconfig` 以便刷新緩存。

需要補充的是，因為動態鏈接庫本身還可能引用其他的庫，那麼一個可執行文件的動態符號鏈接過程可能涉及到多個庫，通過 `readelf -d` 可以打印出該文件直接依賴的庫，而通過 `ldd` 命令則可以打印出所有依賴或者間接依賴的庫。

```
$ ldd test
        linux-gate.so.1 =>  (0xffffe000)
        libc.so.6 => /lib/libc.so.6 (0xb7da2000)
        /lib/ld-linux.so.2 (0xb7efc000)
```

`libc.so.6` 通過 `readelf -d` 就可以看到的，是直接依賴的庫；而 `linux-gate.so.1` 在文件系統中並沒有對應的庫文件，它是一個虛擬的動態鏈接庫，對應進程內存映像的內核部分，更多細節請參考資料[\[11\]][11]; 而 `/lib/ld-linux.so.2` 正好是動態鏈接器，系統需要用它來進行符號重定位。那 `ldd` 是怎麼知道 `/lib/ld-linux.so` 就是該文件的動態鏈接器呢？

那是因為 `ELF` 文件通過專門的節區指定了動態鏈接器，這個節區就是 `.interp` 。

```
$ readelf -x .interp test

Hex dump of section '.interp':
  0x08048114 2f6c6962 2f6c642d 6c696e75 782e736f /lib/ld-linux.so
  0x08048124 2e3200                              .2.
```

可以看到這個節區剛好有字符串 `/lib/ld-linux.so.2`，即 `ld-linux.so` 的絕對路徑。

我們發現，與 `libc.so` 不同的是，`ld-linux.so` 的路徑是絕對路徑，而 `libc.so` 僅僅包含了文件名。原因是：程序被執行時，`ld-linux.so` 將最先被裝載到內存中，沒有其他程序知道去哪裡查找 `ld-linux.so`，所以它的路徑必須是絕對的；當 `ld-linux.so` 被裝載以後，由它來去裝載可執行文件和相關的共享庫，它將根據 `PATH` 變量和 `LD_LIBRARY_PATH` 變量去磁盤上查找它們，因此可執行文件和共享庫都可以不指定絕對路徑。

下面著重介紹動態鏈接器本身。

<span id="toc_23258_14315_8"></span>
### 動態鏈接器（dynamic linker/loader）

Linux 下 `elf` 文件的動態鏈接器是 `ld-linux.so`，即 `/lib/ld-linux.so.2` 。從名字來看和靜態鏈接器 `ld` （`gcc` 默認使用的鏈接器，見參考資料[\[10\]][10]）類似。通過 `man ld-linux` 可以獲取與動態鏈接器相關的資料，包括各種相關的環境變量和文件都有詳細的說明。

對於環境變量，除了上面提到過的 `LD_LIBRARY_PATH` 和 `LD_BIND_NOW` 變量外，還有其他幾個重要參數，比如 `LD_PRELOAD` 用於指定預裝載一些庫，以便替換其他庫中的函數，從而做一些安全方面的處理 [\[6\]][6]，[\[9\]][9]，[\[12\]][12]，而環境變量 `LD_DEBUG` 可以用來進行動態鏈接的相關調試。

對於文件，除了上面提到的 `ld.so.conf` 和 `ld.so.cache` 外，還有一個文件 `/etc/ld.so.preload` 用於指定需要預裝載的庫。

從上一小節中發現有一個專門的節區 `.interp` 存放有動態鏈接器，但是這個節區為什麼叫做 `.interp` （`interpeter`）`呢？因為當 Shell 解釋器或者其他父進程通過 `exec` 啟動我們的程序時，系統會先為 `ld-linux` 創建內存映像，然後把控制權交給 `ld-linux`，之後 `ld-linux` 負責為可執行程序提供運行環境，負責解釋程序的運行，因此 `ld-linux` 也叫做 `dynamic loader` （或 `intepreter`）（關於程序的加載過程請參考資料 [\[13\]][13]）

那麼在 `exec` （）之後和程序指令運行之前的過程是怎樣的呢？ `ld-linux.so` 主要為程序本身創建了內存映像（以下內容摘自資料 [\[8\]][8]），大體過程如下：

- 將可執行文件的內存段添加到進程映像中；
- 把共享目標內存段添加到進程映像中；
- 為可執行文件和它的共享目標（動態鏈接庫）執行重定位操作；
- 關閉用來讀入可執行文件的文件描述符，如果動態鏈接程序收到過這樣的文件描述符的話；
- 將控制轉交給程序，使得程序好像從 `exec()` 直接得到控制

關於第 1 步，在 `ELF` 文件的文件頭中就指定了該文件的入口地址，程序的代碼和數據部分會相繼 `map` 到對應的內存中。而關於可執行文件本身的路徑，如果指定了 `PATH` 環境變量，`ld-linux` 會到 `PATH` 指定的相關目錄下查找。

```
$ readelf -h test | grep Entry
  Entry point address:               0x80482b0
```

對於第 2 步，上一節提到的 `.dynamic` 節區指定了可執行文件依賴的庫名，`ld-linux` （在這裡叫做動態裝載器或程序解釋器比較合適）再從 `LD_LIBRARY_PATH` 指定的路徑中找到相關的庫文件或者直接從 `/etc/ld.so.cache` 庫緩衝中加載相關庫到內存中。（關於進程的內存映像，推薦參考資料 [\[14\]][14]）

對於第 3 步，在前面已提到，如果設置了 `LD_BIND_NOW` 環境變量，這個動作就會在此時發生，否則將會採用 `lazy mode` 方式，即當某個符號被使用時才會進行符號的重定位。不過無論在什麼時候發生這個動作，重定位的過程大體是一樣的（在後面將主要介紹該過程）。

對於第 4 步，這個主要是釋放文件描述符。

對於第 5 步，動態鏈接器把程序控制權交還給程序。

現在關心的主要是第 3 步，即如何進行符號的重定位？下面來探求這個過程。期間會逐步討論到和動態鏈接密切相關的三個數據結構，它們分別是 `ELF` 文件的過程鏈接表、全局偏移表和重定位表，這三個表都是 `ELF` 文件的節區。

<span id="toc_23258_14315_9"></span>
### 過程鏈接表（plt）

從上面的演示發現，還有一個 `printf` 符號的地址沒有確定，它應該在動態鏈接庫 `libc.so` 中定義，需要進行動態鏈接。這裡假設採用 `lazy mode` 方式，即執行到 `printf` 所在位置時才去解析該符號的地址。

假設當前已經執行到了 `printf` 所在位置，即 `call printf`，我們通過 `objdump` 反編譯 `test` 程序的正文段看看。

```
$ objdump -d -s -j .text test | grep printf
 804837c:       e8 1f ff ff ff          call   80482a0 <printf@plt>
```

發現，該地址指向了 `plt` （即過程鏈接表）即地址 `80482a0` 處。下面查看該地址處的內容。

```
$ objdump -D test | grep "80482a0" | grep -v call
080482a0 <printf@plt>:
 80482a0:       ff 25 8c 95 04 08       jmp    *0x804958c
```

發現 `80482a0` 地址對應的是一條跳轉指令，跳轉到 `0x804958c` 地址指向的地址。到底 `0x804958c` 地址本身在什麼地方呢？我們能否從 `.dynamic` 節區（該節區存放了和動態鏈接相關的數據）獲取相關的信息呢？

```
$ readelf -d test

Dynamic section at offset 0x4ac contains 20 entries:
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x8048258
 0x0000000d (FINI)                       0x8048454
 0x00000004 (HASH)                       0x8048148
 0x00000005 (STRTAB)                     0x80481c0
 0x00000006 (SYMTAB)                     0x8048170
 0x0000000a (STRSZ)                      76 (bytes)
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000015 (DEBUG)                      0x0
 0x00000003 (PLTGOT)                     0x8049578
 0x00000002 (PLTRELSZ)                   24 (bytes)
 0x00000014 (PLTREL)                     REL
 0x00000017 (JMPREL)                     0x8048240
 0x00000011 (REL)                        0x8048238
 0x00000012 (RELSZ)                      8 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffe (VERNEED)                    0x8048218
 0x6fffffff (VERNEEDNUM)                 1
 0x6ffffff0 (VERSYM)                     0x804820c
 0x00000000 (NULL)                       0x0
```

發現 `0x8049578` 地址和 `0x804958c` 地址比較近，通過資料 [\[8\]][8] 查到前者正好是 `.got.plt` （即過程鏈接表）對應的全局偏移表的入口地址。難道 `0x804958c` 正好位於 `.got.plt` 節區中？

<span id="toc_23258_14315_10"></span>
### 全局偏移表（got）

現在進入全局偏移表看看，

```
$ readelf -x .got.plt test

Hex dump of section '.got.plt':
  0x08049578 ac940408 00000000 00000000 86820408 ................
  0x08049588 96820408 a6820408                   ........
```

從上述結果可以看出 `0x804958c` 地址（即 `0x08049588+4`）處存放的是 `a6820408`，考慮到我的實驗平臺是 `i386`，字節順序是 `little-endian ` 的，所以實際數值應該是 `080482a6`，也就是說 `*` （`0x804958c`）`的值是 `080482a6`，這個地址剛好是過程鏈接表的最後一項 `call 80482a0<printf@plt>` 中 `80482a0` 地址往後偏移 `6 ` 個字節，容易猜到該地址應該就是 `jmp` 指令的後一條地址。

```
$ objdump -d -d -s -j .plt test |  grep "080482a0 <printf@plt>:" -A 3
080482a0 <printf@plt>:
 80482a0:       ff 25 8c 95 04 08       jmp    *0x804958c
 80482a6:       68 10 00 00 00          push   $0x10
 80482ab:       e9 c0 ff ff ff          jmp    8048270 <_init+0x18>
```

`80482a6` 地址恰巧是一條 `push` 指令，隨後是一條 `jmp` 指令（暫且不管 `push` 指令入棧的內容有什麼意義），執行完 `push` 指令之後，就會跳轉到 `8048270 ` 地址處，下面看看 `8048270 ` 地址處到底有哪些指令。

```
$ objdump -d -d -s -j .plt test | grep -v "jmp    8048270 <_init+0x18>" | grep "08048270" -A 2
08048270 <__gmon_start__@plt-0x10>:
 8048270:       ff 35 7c 95 04 08       pushl  0x804957c
 8048276:       ff 25 80 95 04 08       jmp    *0x8049580
```

同樣是一條入棧指令跟著一條跳轉指令。不過這兩個地址 `0x804957c` 和 `0x8049580` 是連續的，而且都很熟悉，剛好都在 `.got.plt` 表裡頭（從上面我們已經知道 `.got.plt` 的入口是 `0x08049578`）。這樣的話，我們得確認這兩個地址到底有什麼內容。

```
$ readelf -x .got.plt test

Hex dump of section '.got.plt':
  0x08049578 ac940408 00000000 00000000 86820408 ................
  0x08049588 96820408 a6820408                   ........
```

不過，遺憾的是通過 `readelf` 查看到的這兩個地址信息都是 0，它們到底是什麼呢？

現在只能求助參考資料 [\[8\]][8]，該資料的“3.8.5 過程鏈接表”部分在介紹過程鏈接表和全局偏移表相互合作解析符號的過程中的三步涉及到了這兩個地址和前面沒有說明的 `push ` $ `0x10` 指令。

- 在程序第一次創建內存映像時，動態鏈接器為全局偏移表的第二（`0x804957c`）和第三項（`0x8049580`）設置特殊值。
- 原步驟 5。在跳轉到 `08048270 <__gmon_start__@plt-0x10>`，即過程鏈接表的第一項之前，有一條壓入棧指令，即`push $0x10`，0x10是相對於重定位表起始地址的一個偏移地址，這個偏移地址到底有什麼用呢？它應該是提供給動態鏈接器的什麼信息吧？後面再說明。
- 原步驟 6。跳轉到過程鏈接表的第一項之後，壓入了全局偏移表中的第二項（即 `0x804957c` 處），“為動態鏈接器提供了識別信息的機會”（具體是什麼呢？後面會簡單提到，但這個並不是很重要)，然後跳轉到全局偏移表的第三項（`0x8049580`，這一項比較重要），把控制權交給動態鏈接器。

從這三步發現程序運行時地址 `0x8049580` 處存放的應該是動態鏈接器的入口地址，而重定位表 `0x10` 位置處和 `0x804957c` 處應該為動態鏈接器提供瞭解析符號需要的某些信息。

在繼續之前先總結一下過程鏈接表和全局偏移表。上面的操作過程僅僅從“局部”看過了這兩個表，但是並沒有宏觀地看裡頭的內容。下面將宏觀的分析一下， 對於過程鏈接表：

```
$ objdump -d -d -s -j .plt test
08048270 <__gmon_start__@plt-0x10>:
 8048270:       ff 35 7c 95 04 08       pushl  0x804957c
 8048276:       ff 25 80 95 04 08       jmp    *0x8049580
 804827c:       00 00                   add    %al,(%eax)
        ...

08048280 <__gmon_start__@plt>:
 8048280:       ff 25 84 95 04 08       jmp    *0x8049584
 8048286:       68 00 00 00 00          push   $0x0
 804828b:       e9 e0 ff ff ff          jmp    8048270 <_init+0x18>

08048290 <__libc_start_main@plt>:
 8048290:       ff 25 88 95 04 08       jmp    *0x8049588
 8048296:       68 08 00 00 00          push   $0x8
 804829b:       e9 d0 ff ff ff          jmp    8048270 <_init+0x18>

080482a0 <printf@plt>:
 80482a0:       ff 25 8c 95 04 08       jmp    *0x804958c
 80482a6:       68 10 00 00 00          push   $0x10
 80482ab:       e9 c0 ff ff ff          jmp    8048270 <_init+0x18>
```

除了該表中的第一項外，其他各項實際上是類似的。而最後一項 `080482a0 <printf@plt>` 和第一項我們都分析過，因此不難理解其他幾項的作用。過程鏈接表沒有辦法單獨工作，因為它和全局偏移表是關聯的，所以在說明它的作用之前，先從總體上來看一下全局偏移表。

```
$ readelf -x .got.plt test

Hex dump of section '.got.plt':
  0x08049578 ac940408 00000000 00000000 86820408 ................
  0x08049588 96820408 a6820408                   ........
```

比較全局偏移表中 `0x08049584` 處開始的數據和過程鏈接表第二項開始的連續三項中 `push` 指定所在的地址，不難發現，它們是對應的。而 `0x0804958c` 即 `push 0x10` 對應的地址我們剛才提到過（下一節會進一步分析），其他幾項的作用類似，都是跳回到過程鏈接表的 `push` 指令處，隨後就跳轉到過程鏈接表的第一項，以便解析相應的符號（實際上過程鏈接表的第一個表項是進入動態鏈接器，而之前的連續兩個指令則傳送了需要解析的符號等信息）。另外 `0x08049578` 和 `0x08049580` 處分別存放有傳遞給動態鏈接庫的相關信息和動態鏈接器本身的入口地址。但是還有一個地址 `0x08049578`，這個地址剛好是 `.dynamic` 的入口地址，該節區存放了和動態鏈接過程相關的信息，資料 [\[8\]][8] 提到這個表項實際上保留給動態鏈接器自己使用的，以便在不依賴其他程序的情況下對自己進行初始化，所以下面將不再關注該表項。

```
$ objdump -D test | grep 080494ac
080494ac <_DYNAMIC>:
```

<span id="toc_23258_14315_11"></span>
### 重定位表

這裡主要接著上面的 `push 0x10` 指令來分析。通過資料 [\[8\]][8] 發現重定位表包含如何修改其他節區的信息，以便動態鏈接器對某些節區內的符號地址進行重定位（修改為新的地址）。那到底重定位表項提供了什麼樣的信息呢？

- 每一個重定位項有三部分內容，我們重點關注前兩部分。
- 第一部分是 `r_offset`，這裡考慮的是可執行文件，因此根據資料發現，它的取值是被重定位影響（可以說改變或修改）到的存儲單元的虛擬地址。
- 第二部分是 `r_info`，此成員給出要進行重定位的符號表索引（重定位表項引用到的符號表），以及將實施的重定位類型（如何進行符號的重定位）。(Type)。

先來看看重定位表的具體內容，

```
$ readelf -r test

Relocation section '.rel.dyn' at offset 0x238 contains 1 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049574  00000106 R_386_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x240 contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
08049584  00000107 R_386_JUMP_SLOT   00000000   __gmon_start__
08049588  00000207 R_386_JUMP_SLOT   00000000   __libc_start_main
0804958c  00000407 R_386_JUMP_SLOT   00000000   printf
```

僅僅關注和過程鏈接表相關的 `.rel.plt` 部分，`0x10` 剛好是 `1*16+0*1`，即 16 字節，作為重定位表的偏移，剛好對應該表的第三行。發現這個結果中竟然包含了和 `printf` 符號相關的各種信息。不過重定位表中沒有直接指定符號 `printf`，而是根據 `r_info` 部分從動態符號表中計算出來的，注意觀察上述結果中的 `Info` 一列的 1，2，4 和下面結果的 `Num` 列的對應關係。

```
$ readelf -s test | grep ".dynsym" -A 6
Symbol table '.dynsym' contains 5 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     2: 00000000   410 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.0 (2)
     3: 08048474     4 OBJECT  GLOBAL DEFAULT   14 _IO_stdin_used
     4: 00000000    57 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.0 (2)
```

也就是說在執行過程鏈接表中的第一項的跳轉指令（`jmp    *0x8049580`）調用動態鏈接器以後，動態鏈接器因為有了 `push 0x10`，從而可以通過該重定位表項中的 `r_info` 找到對應符號（`printf`）在符號表（`.dynsym`）中的相關信息。

除此之外，符號表中還有 `Offset` （`r_offset`）`以及 `Type` 這兩個重要信息，前者表示該重定位操作後可能影響的地址 `0804958c`，這個地址剛好是 `got` 表項的最後一項，原來存放的是 `push 0x10` 指令的地址。這意味著，該地址處的內容將被修改，而如何修改呢？根據 `Type` 類型 `R_386_JUMP_SLOT`，通過資料 [\[8\]][8] 查找到該類型對應的說明如下（原資料有誤，下面做了修改）：鏈接編輯器創建這種重定位類型主要是為了支持動態鏈接。其偏移地址成員給出過程鏈接表項的位置。動態鏈接器修改全局偏移表項的內容，把控制傳輸給指定符號的地址。

這說明，動態鏈接器將根據該類型對全局偏移表中的最有一項，即 `0804958c` 地址處的內容進行修改，修改為符號的實際地址，即 `printf` 函數在動態鏈接庫的內存映像中的地址。

到這裡，動態鏈接的宏觀過程似乎已經瞭然於心，不過一些細節還是不太清楚。

下面先介紹動態鏈接庫的創建，隱式調用和顯示調用，接著進一步澄清上面還不太清楚的細節，即全局偏移表中第二項到底傳遞給了動態鏈接器什麼信息？第三項是否就是動態鏈接器的地址？並討論通過設置 `LD_BIND_NOW` 而不採用默認的 lazy mode 進行動態鏈接和採用 lazy mode 動態鏈接的區別？

<span id="toc_23258_14315_12"></span>
## 動態鏈接庫的創建和調用

在介紹動態符號鏈接的更多細節之前，先來了解一下動態鏈接庫的創建和兩種使用方法，進而引出符號解析的後臺細節。

<span id="toc_23258_14315_13"></span>
### 創建動態鏈接庫

首先來創建一個簡單動態鏈接庫。

代碼：

```
/* myprintf.c */
#include <stdio.h>

int myprintf(char *str)
{
        printf("%s\n", str);
        return 0;
}
```

```
/* myprintf.h */
#ifndef _MYPRINTF_H
#define _MYPRINTF_H

int myprintf(char *);

#endif
```

演示：

```
$ gcc -c myprintf.c
$ gcc -shared -W1,-soname,libmyprintf.so.0 -o libmyprintf.so.0.0 myprintf.o
$ ln -sf libmyprintf.so.0.0 libmyprintf.so.0
$ ln -fs libmyprintf.so.0 libmyprintf.so
$ ls
libmyprintf.so  libmyprintf.so.0  libmyprintf.so.0.0  myprintf.c  myprintf.h  myprintf.o
```

得到三個文件 `libmyprintf.so`，`libmyprintf.so.0`，`libmyprintf.so.0.0`，這些庫暫且存放在當前目錄下。這裡有一個問題值得關注，那就是為什麼要創建兩個符號鏈接呢？答案是為了在不影響兼容性的前提下升級庫 [\[5\]][5] 。

<span id="toc_23258_14315_14"></span>
### 隱式使用該庫

現在寫一段代碼來使用該庫，調用其中的 `myprintf` 函數，這裡是隱式使用該庫：在代碼中並沒有直接使用該庫，而是通過調用 `myprintf` 隱式地使用了該庫，在編譯引用該庫的可執行文件時需要通過 `-l` 參數指定該庫的名字。

```
/* test.c */
#include <stdio.h>   
#include <myprintf.h>

int main()
{
        myprintf("Hello World");

        return 0;
}
```

編譯：

```
$ gcc -o test test.c -lmyprintf -L./ -I./
```

直接運行 `test`，提示找不到該庫，因為庫的默認搜索路徑裡頭沒有包含當前目錄：

```
$ ./test
./test: error while loading shared libraries: libmyprintf.so: cannot open shared object file: No such file or directory
```

如果指定庫的搜索路徑，則可以運行：

```
$ LD_LIBRARY_PATH=$PWD ./test
Hello World
```

<span id="toc_23258_14315_15"></span>
### 顯式使用庫

`LD_LIBRARY_PATH` 環境變量使得庫可以放到某些指定的路徑下面，而無須在調用程序中顯式的指定該庫的絕對路徑，這樣避免了把程序限制在某些絕對路徑下，方便庫的移動。

雖然顯式調用有不便，但是能夠避免隱式調用搜索路徑的時間消耗，提高效率，除此之外，顯式調用為我們提供了一組函數調用，讓符號的重定位過程一覽無遺。

```
/* test1.c */

#include <dlfcn.h>      /* dlopen, dlsym, dlerror */
#include <stdlib.h>     /* exit */
#include <stdio.h>      /* printf */

#define LIB_SO_NAME     "./libmyprintf.so"
#define FUNC_NAME "myprintf"

typedef int (*func)(char *);

int main(void)
{
        void *h;
        char *e;
        func f;

        h = dlopen(LIB_SO_NAME, RTLD_LAZY);
        if ( !h ) {
                printf("failed load libary: %s\n", LIB_SO_NAME);
                exit(-1);
        }
        f = dlsym(h, FUNC_NAME);
        e = dlerror();
        if (e != NULL) {
                printf("search %s error: %s\n", FUNC_NAME, LIB_SO_NAME);
                exit(-1);
        }
        f("Hello World");

        exit(0);
}
```

演示：

```
$ gcc -o test1 test1.c -ldl
```

這種情況下，無須包含頭文件。從這個代碼中很容易看出符號重定位的過程： 

- 首先通過 `dlopen` 找到依賴庫，並加載到內存中，再返回該庫的 `handle`，通過 `dlopen` 我們可以指定 `RTLD_LAZY` 採用 `lazy mode` 動態鏈接模式，如果採用 `RTLD_NOW` 則和隱式調用時設置 `LD_BIN_NOW` 類似。
- 找到該庫以後就是對某個符號進行重定位，這裡是確定 `myprintf` 函數的地址。
- 找到函數地址以後就可以直接調用該函數了。

關於 `dlopen`，`dlsym` 等後臺工作細節建議參考資料 [\[15\]][15] 。

隱式調用的動態符號鏈接過程和上面類似。下面通過一些實例來確定之前沒有明確的兩個內容：即全局偏移表中的第二項和第三項，並進一步討論lazy mode和非lazy mode的區別。

<span id="toc_23258_14315_16"></span>
## 動態鏈接過程

因為通過 `ELF` 文件，我們就可以確定全局偏移表的位置，因此為了確定全局偏移表位置的第三項和第四項的內容，有兩種辦法：

- 通過 `gdb` 調試。
- 直接在函數內部打印。

因為資料[\[3\]][3]詳細介紹了第一種方法，這裡試著通過第二種方法來確定這兩個地址的值。

```
/**
 * got.c -- get the relative content of the got(global offset table) of an elf file
 */

#include <stdio.h>

#define GOT 0x8049614

int main(int argc, char *argv[])
{
        long got2, got3;
        long old_addr, new_addr;

        got2=*(long *)(GOT+4);
        got3=*(long *)(GOT+8);
        old_addr=*(long *)(GOT+24);

        printf("Hello World\n");

        new_addr=*(long *)(GOT+24);

        printf("got2: 0x%0x, got3: 0x%0x, old_addr: 0x%0x, new_addr: 0x%0x\n",
                                        got2, got3, old_addr, new_addr);

        return 0;
}
```

在寫好上面的代碼後就需要確定全局偏移表的地址，然後把該地址設置為代碼中的宏 `GOT` 。

```
$ make got
$ readelf -d got | grep PLTGOT
 0x00000003 (PLTGOT)                     0x8049614
```

**注**：這裡假設大家用的都是 `i386` 的系統，如果要在 `X86_64` 位系統上要編譯生成 `i386` 上的可執行文件，需要給 `gcc` 傳遞一個 `-m32` 參數，例如：

```
$ gcc -m32 -o got got.c
```

把地址 `0x8049614` 替換到上述代碼中，然後重新編譯運行，查看結果。

```
$ make got
$ Hello World
got2: 0xb7f376d8, got3: 0xb7f2ef10, old_addr: 0x80482da, new_addr: 0xb7e19a20
$ ./got
Hello World
got2: 0xb7f1e6d8, got3: 0xb7f15f10, old_addr: 0x80482da, new_addr: 0xb7e00a20
```

通過兩次運行，發現全局偏移表中的這兩項是變化的，並且 `printf` 的地址對應的 `new_addr` 也是變化的，說明 `libc` 和 `ld-linux` 這兩個庫啟動以後對應的虛擬地址並不確定。因此，無法直接跟蹤到那個地址處的內容，還得藉助調試工具，以便確認它們。

下面重新編譯 `got`，加上 `-g` 參數以便調試，並通過調試確認 `got2`，`got3`，以及調用 `printf` 前後 `printf` 地址的重定位情況。

```
$ gcc -g -o got got.c
$ gdb ./got
(gdb) l
5       #include <stdio.h>
6
7       #define GOT 0x8049614
8
9       int main(int argc, char *argv[])
10      {
11              long got2, got3;
12              long old_addr, new_addr;
13
14              got2=*(long *)(GOT+4);
(gdb) l
15              got3=*(long *)(GOT+8);
16              old_addr=*(long *)(GOT+24);
17
18              printf("Hello World\n");
19
20              new_addr=*(long *)(GOT+24);
21
22              printf("got2: 0x%0x, got3: 0x%0x, old_addr: 0x%0x, new_addr: 0x%0x\n",
23                                              got2, got3, old_addr, new_addr);
24
```

在第一個 `printf` 處設置一個斷點：

```
(gdb) break 18
Breakpoint 1 at 0x80483c3: file got.c, line 18.
```

在第二個 `printf` 處設置一個斷點：

```
(gdb) break 22
Breakpoint 2 at 0x80483dd: file got.c, line 22.
```

運行到第一個 `printf` 之前會停止：

```
(gdb) r
Starting program: /mnt/hda8/Temp/c/program/got

Breakpoint 1, main () at got.c:18
18              printf("Hello World\n");
```

查看執行 `printf` 之前的全局偏移表內容：

```
(gdb) x/8x 0x8049614
0x8049614 <_GLOBAL_OFFSET_TABLE_>:      0x08049548      0xb7f3c6d8      0xb7f33f10      0x080482aa
0x8049624 <_GLOBAL_OFFSET_TABLE_+16>:   0xb7ddbd20      0x080482ca      0x080482da      0x00000000
```

查看 `GOT` 表項的最有一項，發現剛好是 `PLT` 表中 `push` 指令的地址：

```
(gdb) disassemble 0x080482da
Dump of assembler code for function puts@plt:
0x080482d4 <puts@plt+0>:        jmp    *0x804962c
0x080482da <puts@plt+6>:        push   $0x18
0x080482df <puts@plt+11>:       jmp    0x8048294 <_init+24>
```

說明此時還沒有進行進行符號的重定位，不過發現並非 `printf`，而是 `puts(1)`。

接著查看 `GOT` 第三項的內容，剛好是 `dl-linux` 對應的代碼：

```
(gdb) disassemble 0xb7f33f10
Dump of assembler code for function _dl_runtime_resolve:
0xb7f33f10 <_dl_runtime_resolve+0>:     push   %eax
0xb7f33f11 <_dl_runtime_resolve+1>:     push   %ecx
0xb7f33f12 <_dl_runtime_resolve+2>:     push   %edx
```

可通過 `nm /lib/ld-linux.so.2 | grep _dl_runtime_resolve` 進行確認。

然後查看 `GOT` 表第二項處的內容，看不出什麼特別的信息，反編譯時提示無法反編譯：

```
(gdb) x/8x 0xb7f3c6d8
0xb7f3c6d8:     0x00000000      0xb7f39c3d      0x08049548      0xb7f3c9b8
0xb7f3c6e8:     0x00000000      0xb7f3c6d8      0x00000000      0xb7f3c9a4
```

在 `*(0xb7f33f10)` 指向的代碼處設置一個斷點，確認它是否被執行：

```
(gdb) break *(0xb7f33f10)
break *(0xb7f33f10)
Breakpoint 3 at 0xb7f3cf10
(gdb) c
Continuing.

Breakpoint 3, 0xb7f3cf10 in _dl_runtime_resolve () from /lib/ld-linux.so.2
```

繼續運行，直到第二次調用 `printf` ：

```
(gdb)  c
Continuing.
Hello World

Breakpoint 2, main () at got.c:22
22              printf("got2: 0x%0x, got3: 0x%0x, old_addr: 0x%0x, new_addr: 0x%0x\n",
```

再次查看 `GOT` 表項，發現 `GOT` 表的最後一項的值應該被修改：

```
(gdb) x/8x 0x8049614
0x8049614 <_GLOBAL_OFFSET_TABLE_>:      0x08049548      0xb7f3c6d8      0xb7f33f10      0x080482aa
0x8049624 <_GLOBAL_OFFSET_TABLE_+16>:   0xb7ddbd20      0x080482ca      0xb7e1ea20      0x00000000
```

查看 `GOT` 表最後一項，發現變成了 `puts` 函數的代碼，說明進行了符號 `puts` 的重定位（2）：

```
(gdb) disassemble 0xb7e1ea20
Dump of assembler code for function puts:
0xb7e1ea20 <puts+0>:    push   %ebp
0xb7e1ea21 <puts+1>:    mov    %esp,%ebp
0xb7e1ea23 <puts+3>:    sub    $0x1c,%esp
```

通過演示發現一個問題（1）（2），即本來調用的是 `printf`，為什麼會進行 `puts` 的重定位呢？通過 `gcc -S` 參數編譯生成彙編代碼後發現，`gcc` 把 `printf` 替換成了 `puts`，因此不難理解程序運行過程為什麼對 `puts` 進行了重定位。

從演示中不難發現，當符號被使用到時才進行重定位。因為通過調試發現在執行 `printf` 之後，`GOT` 表項的最後一項才被修改為 `printf` （確切的說是 `puts`）的地址。這就是所謂的 `lazy mode` 動態符號鏈接方式。

除此之外，我們容易發現 `GOT` 表第三項確實是 `ld-linux.so` 中的某個函數地址，並且發現在執行 `printf` 語句之前，先進入了 `ld-linux.so` 的 `_dl_runtime_resolve` 函數，而且在它返回之後，`GOT` 表的最後一項才變為 `printf` （`puts`）的地址。

本來打算通過第一個斷點確認第二次調用 `printf` 時不再需要進行動態符號鏈接的，不過因為 `gcc` 把第一個替換成了 `puts`，所以這裡沒有辦法繼續調試。如果想確認這個，你可以通過寫兩個一樣的 `printf` 語句看看。實際上第一次鏈接以後，`GOT` 表的第三項已經修改了，當下次再進入過程鏈接表，並執行 `jmp *(全局偏移表中某一個地址)` 指令時，`*(全局偏移表中某一個地址)` 已經被修改為了對應符號的實際地址，這樣 `jmp` 語句會自動跳轉到符號的地址處運行，執行具體的函數代碼，因此無須再進行重定位。
 
到現在 `GOT` 表中只剩下第二項還沒有被確認，通過資料 [\[3\]][3] 我們發現，該項指向一個 `link_map` 類型的數據，是一個鑑別信息，具體作用對我們來說並不是很重要，如果想了解，請參考資料 [\[16\]][16] 。

下面通過設置 `LD_BIND_NOW` 再運行一下 `got` 程序並查看結果，比較它與默認的動態鏈接方式（`lazy mode`）的異同。

- 設置 `LD_BIND_NOW` 環境變量的運行結果

        $ LD_BIND_NOW=1 ./got
        Hello World
        got2: 0x0, got3: 0x0, old_addr: 0xb7e61a20, new_addr: 0xb7e61a20

- 默認情況下的運行結果

        $ ./got
        Hello World
        got2: 0xb7f806d8, got3: 0xb7f77f10, old_addr: 0x80482da, new_addr: 0xb7e62a20

通過比較容易發現，在非 `lazy mode` （設置 `LD_BIND_NOW` 後）下，程序運行之前符號的地址就已經被確定，即調用 `printf` 之前 `GOT` 表的最後一項已經被確定為了 `printf` 函數對應的地址，即 `0xb7e61a20`，因此在程序運行之後，`GOT` 表的第二項和第三項就保持為 0，因為此時不再需要它們進行符號的重定位了。通過這樣一個比較，就更容易理解 `lazy mode` 的特點了：在用到的時候才解析。

到這裡，符號動態鏈接的細節基本上就已經清楚了。

<span id="toc_23258_14315_17"></span>
## 參考資料

- [LINUX 系統中動態鏈接庫的創建與使用][1]

- [LINUX 動態鏈接庫高級應用][2]

- [ELF 動態解析符號過程(修訂版)][3]

- [如何在 Linux 下調試動態鏈接庫][4]

- [Dissecting shared libraries][5]

- [關於 Linux 和 Unix 動態鏈接庫的安全][6]

- [Linux 系統下解析 ELF 文件 DT_RPATH 後門][7]

- [ELF 文件格式分析][8]

- [緩衝區溢出與注入分析(第二部分：緩衝區溢出和注入實例)][101]
- [Gcc 編譯的背後(第二部分：彙編和鏈接)][100]
- [程序執行的那一剎那][102]

[100]: 02-chapter2.markdown
[101]: 02-chapter5.markdown
[102]: 02-chapter3.markdown

- What is Linux-gate.so.1: [\[1\]][9], [\[2\]][10], [\[3\]][11]

- [Linux 下緩衝區溢出攻擊的原理及對策][12]

- Intel 平臺下 Linux 中 ELF 文件動態鏈接的加載、解析及實例分析 [part1][13], [part2][14]

- [ELF file format and ABI][15]

 [1]: http://www.ccw.com.cn/htm/app/linux/develop/01_8_6_2.asp
 [2]: http://www.vchome.net/tech/dll/dll9.htm
 [3]: http://elfhack.whitecell.org/mydocs/ELF_symbol_resolve_process1.txt
 [4]: http://unix-cd.com/unixcd12/article_5065.html
 [5]: http://www.ibm.com/developerworks/linux/library/l-shlibs.html
 [6]: http://fanqiang.chinaunix.net/safe/system/2007-02-01/4870.shtml
 [7]: http://article.pchome.net/content-323084.html
 [8]: http://162.105.203.48/web/gaikuang/submission/TN05.ELF.Format.Summary.pdf
 [9]: http://www.trilithium.com/johan/2005/08/linux-gate/
 [10]: http://isomerica.net/archives/2007/05/28/what-is-linux-gateso1-and-why-is-it-missing-on-x86-64/
 [11]: http://www.linux010.cn/program/Linux-gateso1-DeHanYi-pcee6103.htm
 [12]: http://www.ibm.com/developerworks/cn/linux/l-overflow/index.html
 [13]: http://www.ibm.com/developerworks/cn/linux/l-elf/part1/index.html
 [14]: http://www.ibm.com/developerworks/cn/linux/l-elf/part2/index.html
 [15]: http://www.x86.org/ftp/manuals/tools/elf.pdf
 [16]: http://www.muppetlabs.com/~breadbox/software/ELF.txt
