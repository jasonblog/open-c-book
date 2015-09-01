# 程序執行的一剎那

-    [什麼是命令行接口](#toc_16100_6031_1)
-    [/bin/bash 是什麼時候啟動的](#toc_16100_6031_2)
    -    [/bin/login](#toc_16100_6031_3)
    -    [/bin/getty](#toc_16100_6031_4)
    -    [/sbin/init](#toc_16100_6031_5)
    -    [命令啟動過程追本溯源](#toc_16100_6031_6)
    -    [誰啟動了 /sbin/init](#toc_16100_6031_7)
-    [/bin/bash 如何處理用戶鍵入的命令](#toc_16100_6031_8)
    -    [預備知識](#toc_16100_6031_9)
    -    [哪種命令先被執行](#toc_16100_6031_10)
    -    [這些特殊字符是如何解析的：`|, >, <, &`](#toc_16100_6031_11)
    -    [/bin/bash 用什麼魔法讓一個普通程序變成了進程](#toc_16100_6031_12)
-    [參考資料](#toc_16100_6031_13)


當我們在 Linux 下的命令行輸入一個命令之後，這背後發生了什麼？

<span id="toc_16100_6031_1"></span>
## 什麼是命令行接口

用戶使用計算機有兩種常見的方式，一種是圖形化的接口（GUI），另外一種則是命令行接口（CLI）。對於圖形化的接口，用戶點擊某個圖標就可啟動後臺的某個程序；對於命令行的接口，用戶鍵入某個程序的名字就可啟動某個程序。這兩者的基本過程是類似的，都需要查找程序文件在磁盤上的位置，加載到內存並通過不同的解釋器進行解析和運行。下面以命令行為例來介紹程序執行一剎那發生的一些事情。

首先來介紹什麼是命令行？命令行就是 `Command Line`，很直觀的概念就是系統啟動後的那個黑屏幕：有一個提示符，並有光標在閃爍的那樣一個終端，一般情況下可以用 `CTRL+ALT+F1-6` 切換到不同的終端；在 GUI 界面下也會有一些偽終端，看上去和系統啟動時的那個終端沒有什麼區別，也會有一個提示符，並有一個光標在閃爍。就提示符和響應用戶的鍵盤輸入而言，它們兩者在功能上是一樣的，實際上它們就是同一個東西，用下面的命令就可以把它們打印出來。

```
$ echo $SHELL # 打印當前SHELL，當前運行的命令行接口程序
/bin/bash
$ echo $$     # 該程序對應進程ID，$$是個特殊的環境變量，它存放了當前進程ID
1481
$ ps -C bash   # 通過PS命令查看
  PID TTY          TIME CMD
 1481 pts/0    00:00:00 bash
```

從上面的操作結果可以看出，當前命令行接口實際上是一個程序，那就是 `/bin/bash`，它是一個實實在在的程序，它打印提示符，接受用戶輸入的命令，分析命令序列並執行然後返回結果。不過 `/bin/bash` 僅僅是當前使用的命令行程序之一，還有很多具有類似功能的程序，比如 `/bin/ash`, `/bin/dash` 等。不過這裡主要來討論 `bash`，討論它自己是怎麼啟動的，它怎麼樣處理用戶輸入的命令等後臺細節？

<span id="toc_16100_6031_2"></span>
## /bin/bash 是什麼時候啟動的

<span id="toc_16100_6031_3"></span>
### /bin/login

先通過 `CTRL+ALT+F1` 切換到一個普通終端下面，一般情況下看到的是 "XXX login: " 提示輸入用戶名，接著是提示輸入密碼，然後呢？就直接登錄到了我們的命令行接口。實際上正是你輸入正確的密碼後，那個程序才把 `/bin/bash` 給啟動了。那是什麼東西提示 "XXX login:" 的呢？正是 `/bin/login` 程序，那 `/bin/login` 程序怎麼知道要啟動 `/bin/bash`，而不是其他的 `/bin/dash` 呢？

`/bin/login` 程序實際上會檢查我們的 `/etc/passwd` 文件，在這個文件裡頭包含了用戶名、密碼和該用戶的登錄 Shell。密碼和用戶名匹配用戶的登錄，而登錄 Shell 則作為用戶登錄後的命令行程序。看看 `/etc/passwd` 中典型的這麼一行：

```
$ cat /etc/passwd | grep falcon
falcon:x:1000:1000:falcon,,,:/home/falcon:/bin/bash
```

這個是我用的帳號的相關信息哦，看到最後一行沒？`/bin/bash`，這正是我登錄用的命令行解釋程序。至於密碼呢，看到那個 **`x`** 沒？這個 `x` 說明我的密碼被保存在另外一個文件裡頭 `/etc/shadow`，而且密碼是經過加密的。至於這兩個文件的更多細節，看手冊吧。

我們怎麼知道剛好是 `/bin/login` 打印了 "XXX login" 呢？現在回顧一下很早以前學習的那個 `strace` 命令。我們可以用 `strace` 命令來跟蹤 `/bin/login` 程序的執行。

跟上面一樣，切換到一個普通終端，並切換到 Root 用戶，用下面的命令：

```
$ strace -f -o strace.out /bin/login
```

退出以後就可以打開 `strace.out` 文件，看看到底執行了哪些文件，讀取了哪些文件。從中可以看到正是 `/bin/login` 程序用 `execve` 調用了 `/bin/bash` 命令。通過後面的演示，可以發現 `/bin/login` 只是在子進程裡頭用 `execve` 調用了 `/bin/bash`，因為在啟動 `/bin/bash` 後，可以看到 `/bin/login` 並沒有退出。

<span id="toc_16100_6031_4"></span>
### /bin/getty

那 `/bin/login` 又是怎麼起來的呢？

下面再來看一個演示。先在一個可以登陸的終端下執行下面的命令。

```
$ getty 38400 tty8 linux
```

`getty` 命令停留在那裡，貌似等待用戶的什麼操作，現在切回到第 8 個終端，是不是看到有 "XXX login:" 的提示了。輸入用戶名並登錄，之後退出，回到第一個終端，發現 `getty` 命令已經退出。

類似地，也可以用 `strace` 命令來跟蹤 `getty` 的執行過程。在第一個終端下切換到 Root 用戶。執行如下命令：

```
$ strace -f -o strace.out getty 38400 tty8 linux
```

同樣在 `strace.out` 命令中可以找到該命令的相關啟動細節。比如，可以看到正是 `getty` 程序用 `execve` 系統調用執行了 `/bin/login` 程序。這個地方，`getty` 是在自己的主進程裡頭直接執行了 `/bin/login`，這樣 `/bin/login` 將把 `getty` 的進程空間替換掉。

<span id="toc_16100_6031_5"></span>
### /sbin/init

這裡涉及到一個非常重要的東西：`/sbin/init`，通過 `man init` 命令可以查看到該命令的作用，它可是“萬物之王”（init  is  the  parent of all processes on the system）哦。它是 Linux 系統默認啟動的第一個程序，負責進行 Linux 系統的一些初始化工作，而這些初始化工作的配置則是通過 `/etc/inittab` 來做的。那麼來看看 `/etc/inittab` 的一個簡單的例子吧，可以通過 `man inittab` 查看相關幫助。

需要注意的是，在較新版本的 Ubuntu 和 Fedora 等發行版中，一些新的 `init` 程序，比如 `upstart` 和 `systemd` 被開發出來用於取代 `System V init`，它們可能放棄了對 `/etc/inittab` 的使用，例如 `upstart` 會讀取 `/etc/init/` 下的配置，比如 `/etc/init/tty1.conf`，但是，基本的配置思路還是類似 `/etc/inittab`，對於 `upstart` 的 `init` 配置，這裡不做介紹，請通過 `man 5 init` 查看幫助。

配置文件 `/etc/inittab` 的語法非常簡單，就是下面一行的重複，

```
id:runlevels:action:process
```

-   `id` 就是一個唯一的編號，不用管它，一個名字而言，無關緊要。

-   `runlevels` 是運行級別，這個還是比較重要的，理解運行級別的概念很有必要，它可以有如下的取值：

        0 is halt.
        1 is single-user.
        2-5 are multi-user.
        6 is reboot.

    不過，真正在配置文件裡頭用的是 `1-5` 了，而 `0` 和 `6` 非常特別，除了用它作為 `init` 命令的參數關機和重啟外，似乎沒有哪個“傻瓜”把它寫在系統的配置文件裡頭，讓系統啟動以後就關機或者重啟。`1` 代表單用戶，而 `2-5` 則代表多用戶。對於 `2-5` 可能有不同的解釋，比如在 Slackware 12.0 上，`2,3,5` 被用來作為多用戶模式，但是默認不啟動 X windows （GUI接口），而 `4` 則作為啟動 X windows 的運行級別。

-   `action` 是動作，它也有很多選擇，我們關心幾個常用的
    
-   `initdefault`：用來指定系統啟動後進入的運行級別，通常在 `/etc/inittab` 的第一條配置，如：

        id:3:initdefault:

    這個說明默認運行級別是 3，即多用戶模式，但是不啟動 X window 的那種。

-   `sysinit`：指定那些在系統啟動時將被執行的程序，例如：

        si:S:sysinit:/etc/rc.d/rc.S

    在 `man inittab` 中提到，對於 `sysinit`，`boot` 等動作，`runlevels` 選項是不用管的，所以可以很容易解讀這條配置：它的意思是系統啟動時將默認執行 `/etc/rc.d/rc.S` 文件，在這個文件裡可直接或者間接地執行想讓系統啟動時執行的任何程序，完成系統的初始化。

-   `wait`：當進入某個特別的運行級別時，指定的程序將被執行一次，`init` 將等到它執行完成，例如：

        rc:2345:wait:/etc/rc.d/rc.M

    這個說明無論是進入運行級別 2，3，4，5 中哪一個，`/etc/rc.d/rc.M` 將被執行一次，並且有 `init` 等待它執行完成。

-   `ctrlaltdel`，當 `init` 程序接收到 `SIGINT` 信號時，某個指定的程序將被執行，我們通常通過按下 `CTRL+ALT+DEL`，這個默認情況下將給 `init` 發送一個 `SIGINT` 信號。

    如果我們想在按下這幾個鍵時，系統重啟，那麼可以在 `/etc/inittab` 中寫入：

        ca::ctrlaltdel:/sbin/shutdown -t5 -r now

-   `respawn`：這個指定的進程將被重啟，任何時候當它退出時。這意味著沒有辦法結束它，除非 `init` 自己結束了。例如：

        c1:1235:respawn:/sbin/agetty 38400 tty1 linux

    這一行的意思非常簡單，就是系統運行在級別 1，2，3，5 時，將默認執行 `/sbin/agetty` 程序（這個類似於上面提到的 `getty` 程序），這個程序非常有意思，就是無論什麼時候它退出，`init` 將再次啟動它。這個有幾個比較有意思的問題：

* 在 Slackware 12.0 下，當默認運行級別為 4 時，只有第 6 個終端可以用。原因是什麼呢？因為類似上面的配置，因為那裡只有 `1235`，而沒有 `4`，這意味著當系統運行在第 `4` 級別時，其他終端下的 `/sbin/agetty` 沒有啟動。所以，如果想讓其他終端都可以用，把 `1235` 修改為 `12345` 即可。
* 另外一個有趣的問題就是：正是 `init` 程序在讀取這個配置行以後啟動了 `/sbin/agetty`，這就是 `/sbin/agetty` 的祕密。
* 還有一個問題：無論退出哪個終端，那個 "XXX login:" 總是會被打印，原因是 `respawn` 動作有趣的性質，因為它告訴 `init`，無論 `/sbin/agetty` 什麼時候退出，重新把它啟動起來，那跟 "XXX login:" 有什麼關係呢？從前面的內容，我們發現正是 `/sbin/getty` （同 `agetty`）啟動了 `/bin/login`，而 `/bin/login` 又啟動了 `/bin/bash`，即我們的命令行程序。

<span id="toc_16100_6031_6"></span>
### 命令啟動過程追本溯源

而 `init` 程序作為“萬物之王”，它是所有進程的“父”（也可能是祖父……）進程，那意味著其他進程最多隻能是它的兒子進程。而這個子進程是怎麼創建的，`fork` 調用，而不是之前提到的 `execve` 調用。前者創建一個子進程，後者則會覆蓋當前進程。因為我們發現 `/sbin/getty` 運行時，`init` 並沒有退出，因此可以判斷是 `fork` 調用創建一個子進程後，才通過 `execve` 執行了 `/sbin/getty`。

因此，可以總結出這麼一個調用過程：

```
     fork     execve         execve         fork           execve
init --> init --> /sbin/getty --> /bin/login --> /bin/login --> /bin/bash
```

這裡的 `execve` 調用以後，後者將直接替換前者，因此當鍵入 `exit` 退出 `/bin/bash` 以後，也就相當於 `/sbin/getty` 都已經結束了，因此最前面的 `init` 程序判斷 `/sbin/getty` 退出了，又會創建一個子進程把 `/sbin/getty` 啟動，進而又啟動了 `/bin/login`，又看到了那個 "XXX login:"。

通過 `ps` 和 `pstree` 命令看看實際情況是不是這樣，前者打印出進程的信息，後者則打印出調用關係。

```
$ ps -ef | egrep "/sbin/init|/sbin/getty|bash|/bin/login"
root         1     0  0 21:43 ?        00:00:01 /sbin/init
root      3957     1  0 21:43 tty4     00:00:00 /sbin/getty 38400 tty4
root      3958     1  0 21:43 tty5     00:00:00 /sbin/getty 38400 tty5
root      3963     1  0 21:43 tty3     00:00:00 /sbin/getty 38400 tty3
root      3965     1  0 21:43 tty6     00:00:00 /sbin/getty 38400 tty6
root      7023     1  0 22:48 tty1     00:00:00 /sbin/getty 38400 tty1
root      7081     1  0 22:51 tty2     00:00:00 /bin/login --
falcon    7092  7081  0 22:52 tty2     00:00:00 -bash
```

上面的結果已經過濾了一些不相干的數據。從上面的結果可以看到，除了 `tty2` 被替換成 `/bin/login` 外，其他終端都運行著 `/sbin/getty`，說明終端 2 上的進程是 `/bin/login`，它已經把 `/sbin/getty` 替換掉，另外，我們看到 `-bash` 進程的父進程是 `7081` 剛好是 `/bin/login` 程序，這說明 `/bin/login` 啟動了 `-bash`，但是它並沒有替換掉 `/bin/login`，而是成為了 `/bin/login` 的子進程，這說明 `/bin/login` 通過 `fork` 創建了一個子進程並通過 `execve` 執行了 `-bash`（後者通過 `strace`跟蹤到）。而 `init` 呢，其進程 ID 是 1，是 `/sbin/getty` 和 `/bin/login` 的父進程，說明 `init` 啟動或者間接啟動了它們。下面通過 `pstree` 來查看調用樹，可以更清晰地看出上述關係。

```
$ pstree | egrep "init|getty|\-bash|login"
init-+-5*[getty]
     |-login---bash
     |-xfce4-terminal-+-bash-+-grep
```

結果顯示 `init` 是 5 個 `getty` 程序，`login` 程序和 `xfce4-terminal` 的父進程，而後兩者則是 `bash` 的父進程，另外我們執行的 `grep` 命令則在 `bash` 上運行，是 `bash` 的子進程，這個將是我們後面關心的問題。

從上面的結果發現，`init` 作為所有進程的父進程，它的父進程 ID 饒有興趣的是 0，它是怎麼被啟動的呢？誰才是真正的“造物主”？

<span id="toc_16100_6031_7"></span>
### 誰啟動了 /sbin/init

如果用過 `Lilo` 或者 `Grub` 這些操作系統引導程序，可能會用到 Linux 內核的一個啟動參數 `init`，當忘記密碼時，可能會把這個參數設置成 `/bin/bash`，讓系統直接進入命令行，而無須輸入帳號和密碼，這樣就可以方便地把登錄密碼修改掉。

這個 `init` 參數是個什麼東西呢？通過 `man bootparam` 會發現它的祕密，`init` 參數正好指定了內核啟動後要啟動的第一個程序，而如果沒有指定該參數，內核將依次查找 `/sbin/init`，`/etc/init`，`/bin/init`，`/bin/sh`，如果找不到這幾個文件中的任何一個，內核就要恐慌（panic）了，並掛（hang）在那裡一動不動了（注：如果 `panic=timeout` 被傳遞給內核並且 `timeout` 大於 0，那麼就不會掛住而是重啟）。

因此 `/sbin/init` 就是 Linux 內核啟動的。而 Linux 內核呢？是通過 `Lilo` 或者 `Grub` 等引導程序啟動的，`Lilo` 和 `Grub` 都有相應的配置文件，一般對應 `/etc/lilo.conf` 和 `/boot/grub/menu.lst`，通過這些配置文件可以指定內核映像文件、系統根目錄所在分區、啟動選項標籤等信息，從而能夠讓它們順利把內核啟動起來。

那 `Lilo` 和 `Grub` 本身又是怎麼被運行起來的呢？有了解 MBR 不？MBR 就是主引導扇區，一般情況下這裡存放著 `Lilo` 和 `Grub` 的代碼，而誰知道正好是這裡存放了它們呢？BIOS，如果你用光盤安裝過操作系統的話，那麼應該修改過 `BIOS` 的默認啟動設置，通過設置可以讓系統從光盤、硬盤、U 盤甚至軟盤啟動。正是這裡的設置讓 BIOS 知道了 MBR 處的代碼需要被執行。

那 BIOS 又是什麼時候被起來的呢？處理器加電後有一個默認的起始地址，一上電就執行到了這裡，再之前就是開機鍵按鍵後的上電時序。

更多系統啟動的細節，看看 `man boot-scripts` 吧。

到這裡，`/bin/bash` 的神祕面紗就被揭開了，它只是系統啟動後運行的一個程序而已，只不過這個程序可以響應用戶的請求，那它到底是如何響應用戶請求的呢？

<span id="toc_16100_6031_8"></span>
## /bin/bash 如何處理用戶鍵入的命令

<span id="toc_16100_6031_9"></span>
### 預備知識

在執行磁盤上某個程序時，通常不會指定這個程序文件的絕對路徑，比如要執行 `echo` 命令時，一般不會輸入 `/bin/echo`，而僅僅是輸入 `echo`。那為什麼這樣 `bash` 也能夠找到 `/bin/echo` 呢？原因是 Linux 操作系統支持這樣一種策略：Shell 的一個環境變量 `PATH` 裡頭存放了程序的一些路徑，當 Shell 執行程序時有可能去這些目錄下查找。`which` 作為 Shell（這裡特指 `bash`）的一個內置命令，如果用戶輸入的命令是磁盤上的某個程序，它會返回這個文件的全路徑。

有三個東西和終端的關係很大，那就是標準輸入、標準輸出和標準錯誤，它們是三個文件描述符，一般對應描述符 0，1，2。在 C 語言程序裡，我們可以把它們當作文件描述符一樣進行操作。在命令行下，則可以使用重定向字符`>，<`等對它們進行操作。對於標準輸出和標準錯誤，都默認輸出到終端，對於標準輸入，也同樣默認從終端輸入。

<span id="toc_16100_6031_10"></span>
### 哪種命令先被執行

在 C 語言裡頭要寫一段輸入字符串的命令很簡單，調用 `scanf` 或者 `fgets` 就可以。這個在 `bash` 裡頭應該是類似的。但是它獲取用戶的命令以後，如何分析命令，如何響應不同的命令呢？

首先來看看 `bash` 下所謂的命令，用最常見的 `test` 來作測試。

- 字符串被解析成命令

    隨便鍵入一個字符串 `test1`， `bash` 發出響應，告知找不到這個程序：

        $ test1
        bash: test1: command not found

- 內置命令

    而當鍵入 `test` 時，看不到任何輸出，唯一響應是，新命令提示符被打印了：

        $ test
        $

    查看 `test` 這個命令的類型，即查看 `test` 將被如何解釋， `type` 告訴我們 `test` 是一個內置命令，如果沒有理解錯， `test` 應該是利用諸如 `case "test": do something;break;` 這樣的機制實現的，具體如何實現可以查看 `bash` 源代碼。

        $ type test
        test is a shell builtin

- 外部命令

    這裡通過 `which` 查到 `/usr/bin` 下有一個 `test` 命令文件，在鍵入 `test` 時，到底哪一個被執行了呢？

        $ which test
        /usr/bin/test

    執行這個呢？也沒什麼反應，到底誰先被執行了？

        $ /usr/bin/test

    從上述演示中發現一個問題？如果輸入一個命令，這個命令要麼就不存在，要麼可能同時是 Shell 的內置命令、也有可能是磁盤上環境變量 `PATH` 所指定的目錄下的某個程序文件。

    考慮到 `test` 內置命令和 `/usr/bin/test` 命令的響應結果一樣，我們無法知道哪一個先被執行了，怎麼辦呢？把 `/usr/bin/test` 替換成一個我們自己的命令，並讓它打印一些信息(比如 `hello, world!` )，這樣我們就知道到底誰被執行了。寫完程序，編譯好，命名為 `test` 放到 `/usr/bin` 下（記得備份原來那個）。開始測試：

    鍵入 `test` ，還是沒有效果：

        $ test
        $

    而鍵入絕對路徑呢，則打印了 `hello, world!` 誒，那默認情況下肯定是內置命令先被執行了：

        $ /usr/bin/test
        hello, world!

    由上述實驗結果可見，內置命令比磁盤文件中的程序優先被 `bash` 執行。原因應該是內置命令避免了不必要的 `fork/execve` 調用，對於採用類似算法實現的功能，內置命令理論上有更高運行效率。

    下面看看更多有趣的內容，鍵盤鍵入的命令還有可能是什麼呢？因為 `bash` 支持別名（`alias`）和函數（`function`），所以還有可能是別名和函數，另外，如果 `PATH` 環境變量指定的不同目錄下有相同名字的程序文件，那到底哪個被優先找到呢？

    下面再作一些實驗，

- 別名

    把 `test` 命名為 `ls -l` 的別名，再執行 `test` ，竟然執行了 `ls -l` ，說明別名（`alias`）比內置命令（`builtin`）更優先：

        $ alias test="ls -l"
        $ test
        total 9488
        drwxr-xr-x 12 falcon falcon    4096 2008-02-21 23:43 bash-3.2
        -rw-r--r--  1 falcon falcon 2529838 2008-02-21 23:30 bash-3.2.tar.gz

- 函數

    定義一個名叫 `test` 的函數，執行一下，發現，還是執行了 `ls -l` ，說明 `function` 沒有 `alias` 優先級高：

        $ function test { echo "hi, I'm a function"; }
        $ test
        total 9488
        drwxr-xr-x 12 falcon falcon    4096 2008-02-21 23:43 bash-3.2
        -rw-r--r--  1 falcon falcon 2529838 2008-02-21 23:30 bash-3.2.tar.gz

    把別名給去掉（`unalias`），現在執行的是函數，說明函數的優先級比內置命令也要高：

        $ unalias test
        $ test
        hi, I'm a function

    如果在命令之前跟上 `builtin` ，那麼將直接執行內置命令：

        $ builtin test

    要去掉某個函數的定義，這樣就可以：

        $ unset test

通過這個實驗我們得到一個命令的別名（`alias`）、函數（`function`），內置命令（`builtin`）和程序（`program`）的執行優先次序：

        先    alias --> function --> builtin --> program   後

實際上， `type` 命令會告訴我們這些細節， `type -a` 會按照 `bash` 解析的順序依次打印該命令的類型，而 `type -t` 則會給出第一個將被解析的命令的類型，之所以要做上面的實驗，是為了讓大家加印象。

```
$ type -a test
test is a shell builtin
test is /usr/bin/test
$ alias test="ls -l"
$ function test { echo "I'm a function"; }
$ type -a test
test is aliased to `ls -l'
test is a function
test ()
{
    echo "I'm a function"
}
test is a shell builtin
test is /usr/bin/test
$ type -t test
alias
```

下面再看看 `PATH` 指定的多個目錄下有同名程序的情況。再寫一個程序，打印 `hi, world!`，以示和 `hello, world!` 的區別，放到 `PATH` 指定的另外一個目錄 `/bin` 下，為了保證測試的說服力，再寫一個放到另外一個叫 `/usr/local/sbin` 的目錄下。

先看看 `PATH` 環境變量，確保它有 `/usr/bin`，`/bin` 和 `/usr/local/sbin` 這幾個目錄，然後通過 `type -P`（`-P` 參數強制到 `PATH` 下查找，而不管是別名還是內置命令等，可以通過 `help type` 查看該參數的含義）查看，到底哪個先被執行。

$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
$ type -P test
/usr/local/sbin/test

如上可以看到 `/usr/local/sbin` 下的先被找到。

把 `/usr/local/sbin/test` 下的給刪除掉，現在 `/usr/bin` 下的先被找到：

```
$ rm /usr/local/sbin/test
$ type -P test
/usr/bin/test
```

`type -a` 也顯示類似的結果：

```
$ type -a test
test is aliased to `ls -l'
test is a function
test ()
{
    echo "I'm a function"
}
test is a shell builtin
test is /usr/bin/test
test is /bin/test
```

因此，可以找出這麼一個規律： Shell 從 `PATH` 列出的路徑中依次查找用戶輸入的命令。考慮到程序的優先級最低，如果想優先執行磁盤上的程序文件 `test` 呢？那麼就可以用 `test -P` 找出這個文件並執行就可以了。

補充：對於 Shell 的內置命令，可以通過 `help command` 的方式獲得幫助，對於程序文件，可以查看用戶手冊（當然，這個需要安裝，一般叫做 `xxx-doc`）， `man command` 。

<span id="toc_16100_6031_11"></span>
### 這些特殊字符是如何解析的：`|, >, <, &`

在命令行上，除了輸入各種命令以及一些參數外，比如上面 `type` 命令的各種參數 `-a`，`-P` 等，對於這些參數，是傳遞給程序本身的，非常好處理，比如 `if` ， `else` 條件分支或者 `switch`，`case` 都可以處理。當然，在 `bash` 裡頭可能使用專門的參數處理函數 `getopt` 和 `getopt_long` 來處理它們。

而 `|` ， `>` ， `<` ， `&` 等字符，則比較特別， Shell 是怎麼處理它們的呢？它們也被傳遞給程序本身嗎？可我們的程序內部一般都不處理這些字符的，所以應該是 Shell 程序自己解析了它們。

先來看看這幾個字符在命令行的常見用法，


`<` 字符表示：把 `test.c` 文件重定向為標準輸入，作為 `cat` 命令輸入，而 `cat` 默認輸出到標準輸出：

```
$ cat < ./test.c
#include <stdio.h>

int main(void)
{
        printf("hi, myself!\n");
        return 0;
}
```

`>` 表示把標準輸出重定向為文件 `test_new.c` ，結果內容輸出到 `test_new.c` ：

```
$ cat < ./test.c > test_new.c
```

對於 `>` ， `<` ， `>>` ， `<<` ， `<>` 我們都稱之為重定向（`redirect`）， Shell 到底是怎麼進行所謂的“重定向”的呢？

這主要歸功於 `dup/fcntl` 等函數，它們可以實現：複製文件描述符，讓多個文件描述符共享同一個文件表項。比如，當把文件 `test.c` 重定向為標準輸入時。假設之前用以打開 `test.c` 的文件描述符是 5 ，現在就把 5 複製為了 0 ，這樣當 `cat` 試圖從標準輸入讀出內容時，也就訪問了文件描述符 5 指向的文件表項，接著讀出了文件內容。輸出重定向與此類似。其他的重定向，諸如 `>>` ， `<<` ， `<>` 等雖然和 `>` ， `<` 的具體實現功能不太一樣，但本質是一樣的，都是文件描述符的複製，只不過可能對文件操作有一些附加的限制，比如 `>>` 在輸出時追加到文件末尾，而 `>` 則會從頭開始寫入文件，前者意味著文件的大小會增長，而後者則意味文件被重寫。

那麼 `|` 呢？ `|` 被形象地稱為“管道”，實際上它就是通過 C 語言裡頭的無名管道來實現的。先看一個例子，

```
$ cat < ./test.c  | grep hi
        printf("hi, myself!\n");
```

在這個例子中， `cat` 讀出了 `test.c` 文件中的內容，並輸出到標準輸出上，但是實際上輸出的內容卻只有一行，原因是這個標準輸出被“接到”了 `grep` 命令的標準輸入上，而 `grep` 命令只打印了包含 “hi” 字符串的一行。

這是怎麼被“接”上的。 `cat` 和 `grep` 作為兩個單獨的命令，它們本身沒有辦法把兩者的輸入和輸出“接”起來。這正是 Shell 自己的“傑作”，它通過 C 語言裡頭的 `pipe` 函數創建了一個管道（一個包含兩個文件描述符的整形數組，一個描述符用於寫入數據，一個描述符用於讀入數據），並且通過 `dup/fcntl` 把 `cat` 的輸出複製到了管道的輸入，而把管道的輸出則複製到了 `grep` 的輸入。這真是一個奇妙的想法。

那 `&` 呢？當你在程序的最後跟上這個奇妙的字符以後就可以接著做其他事情了，看看效果：

```
$ sleep 50 & #讓程序在後臺運行
[1] 8261
```

提示符被打印出來，可以輸入東西，讓程序到前臺運行，無法輸入東西了，按下 `CTRL+Z` ，再讓程序到後臺運行：

```
$ fg %1
sleep 50

[1]+  Stopped                 sleep 50
```

實際上 `&` 正是 `Shell ` 支持作業控制的表徵，通過作業控制，用戶在命令行上可以同時作幾個事情（把當前不做的放到後臺，用 `&` 或者 `CTRL+Z` 或者 `bg`）並且可以自由地選擇當前需要執行哪一個（用 `fg` 調到前臺）。這在實現時應該涉及到很多東西，包括終端會話（`session`）、終端信號、前臺進程、後臺進程等。而在命令的後面加上 `&` 後，該命令將被作為後臺進程執行，後臺進程是什麼呢？這類進程無法接收用戶發送給終端的信號（如 `SIGHUP` ，`SIGQUIT` ，`SIGINT`），無法響應鍵盤輸入（被前臺進程佔用著），不過可以通過 `fg` 切換到前臺而享受作為前臺進程具有的特權。

因此，當一個命令被加上 `&` 執行後，Shell 必須讓它具有後臺進程的特徵，讓它無法響應鍵盤的輸入，無法響應終端的信號（意味忽略這些信號），並且比較重要的是新的命令提示符得打印出來，並且讓命令行接口可以繼續執行其他命令，這些就是 Shell 對 `&` 的執行動作。

還有什麼神祕的呢？你也可以寫自己的 Shell 了，並且可以讓內核啟動後就執行它 `l` ，在 `lilo` 或者 `grub` 的啟動參數上設置 `init=/path/to/your/own/shell/program` 就可以。當然，也可以把它作為自己的登錄 Shell ，只需要放到 `/etc/passwd` 文件中相應用戶名所在行的最後就可以。不過貌似到現在還沒介紹 Shell 是怎麼執行程序，是怎樣讓程序變成進程的，所以繼續。

<span id="toc_16100_6031_12"></span>
### /bin/bash 用什麼魔法讓一個普通程序變成了進程

當我們從鍵盤鍵入一串命令，Shell 奇妙地響應了，對於內置命令和函數，Shell 自身就可以解析了（通過 `switch` ，`case` 之類的 C 語言語句）。但是，如果這個命令是磁盤上的一個文件呢。它找到該文件以後，怎麼執行它的呢？

還是用 `strace` 來跟蹤一個命令的執行過程看看。

```
$ strace -f -o strace.log /usr/bin/test
hello, world!
$ cat strace.log | sed -ne "1p"   #我們對第一行很感興趣
8445  execve("/usr/bin/test", ["/usr/bin/test"], [/* 33 vars */]) = 0
```

從跟蹤到的結果的第一行可以看到 `bash` 通過 `execve` 調用了 `/usr/bin/test` ，並且給它傳了 33 個參數。這 33 個 `vars` 是什麼呢？看看 `declare -x` 的結果（這個結果只有 32 個，原因是 `vars` 的最後一個變量需要是一個結束標誌，即 `NULL`）。

```
$ declare -x | wc -l   #declare -x聲明的環境變量將被導出到子進程中
32
$ export TEST="just a test"   #為了認證declare -x和之前的vars的個數的關係，再加一個
$ declare -x | wc -l
33
$ strace -f -o strace.log /usr/bin/test   #再次跟蹤，看看這個關係
hello, world!
$ cat strace.log | sed -ne "1p"
8523  execve("/usr/bin/test", ["/usr/bin/test"], [/* 34 vars */]) = 0
```

通過這個演示發現，當前 Shell 的環境變量中被設置為 `export` 的變量被複制到了新的程序裡頭。不過雖然我們認為 Shell 執行新程序時是在一個新的進程裡頭執行的，但是 `strace` 並沒有跟蹤到諸如 `fork` 的系統調用（可能是 `strace` 自己設計的時候並沒有跟蹤 `fork` ，或者是在 `fork` 之後才跟蹤）。但是有一個事實我們不得不承認：當前 Shell 並沒有被新程序的進程替換，所以說 Shell 肯定是先調用 `fork` （也有可能是 `vfork`）創建了一個子進程，然後再調用 `execve` 執行新程序的。如果你還不相信，那麼直接通過 `exec` 執行新程序看看，這個可是直接把當前 Shell 的進程替換掉的。

```
exec /usr/bin/test
```

該可以看到當前 Shell “譁”（聽不到，突然沒了而已）的一下就沒有了。

下面來模擬一下 Shell 執行普通程序。 `multiprocess` 相當於當前 Shell ，而 `/usr/bin/test` 則相當於通過命令行傳遞給 Shell 的一個程序。這裡是代碼：

```
/* multiprocess.c */
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>     /* sleep, fork, _exit */

int main()
{
	int child;
	int status;

	if( (child = fork()) == 0) {    /* child */
		printf("child: my pid is %d\n", getpid());
		printf("child: my parent's pid is %d\n", getppid());
		execlp("/usr/bin/test","/usr/bin/test",(char *)NULL);;
	} else if(child < 0){   	/* error */
        	printf("create child process error!\n");
        	_exit(0);
	}                                               	/* parent */
	printf("parent: my pid is %d\n", getpid());
	if ( wait(&status) == child ) {
		printf("parent: wait for my child exit successfully!\n");
	}
}
```

運行看看，

```
$ make multiprocess
$ ./multiprocess
child: my pid is 2251
child: my parent's pid is 2250
hello, world!
parent: my pid is 2250
parent: wait for my child exit successfully!
```

從執行結果可以看出，`/usr/bin/test` 在 `multiprocess` 的子進程中運行並不干擾父進程，因為父進程一直等到了 `/usr/bin/test` 執行完成。

再回頭看看代碼，你會發現 `execlp` 並沒有傳遞任何環境變量信息給 `/usr/bin/test` ，到底是怎麼把環境變量傳送過去的呢？通過 `man exec` 我們可以看到一組 `exec` 的調用，在裡頭並沒有發現 `execve` ，但是通過 `man execve` 可以看到該系統調用。實際上 `exec` 的那一組調用都只是 `libc` 庫提供的，而 `execve` 才是真正的系統調用，也就是說無論使用 `exec` 調用中的哪一個，最終調用的都是 `execve` ，如果使用 `execlp` ，那麼 `execlp` 將通過一定的處理把參數轉換為 `execve` 的參數。因此，雖然我們沒有傳遞任何環境變量給 `execlp` ，但是默認情況下，`execlp` 把父進程的環境變量複製給了子進程，而這個動作是在 `execlp` 函數內部完成的。

現在，總結一下 `execve` ，它有有三個參數，

 `- ` 第一個是程序本身的絕對路徑，對於剛才使用的 `execlp` ，我們沒有指定路徑，這意味著它會設法到 `PATH` 環境變量指定的路徑下去尋找程序的全路徑。
 `- ` 第二個參數是一個將傳遞給被它執行的程序的參數數組指針。正是這個參數把我們從命令行上輸入的那些參數，諸如 `grep` 命令的 `-v` 等傳遞給了新程序，可以通過 `main` 函數的第二個參數 `char ` * `argv[]` 獲得這些內容。
 `- ` 第三個參數是一個將傳遞給被它執行的程序的環境變量，這些環境變量也可以通過 `main` 函數的第三個變量獲取，只要定義一個 `char ` * `env[]` 就可以了，只是通常不直接用它罷了，而是通過另外的方式，通過 `extern char ` ** `environ` 全局變量（環境變量表的指針）或者 `getenv` 函數來獲取某個環境變量的值。

當然，實際上，當程序被 `execve` 執行後，它被加載到了內存裡，包括程序的各種指令、數據以及傳遞給它的各種參數、環境變量等都被存放在系統分配給該程序的內存空間中。

我們可以通過 `/proc/<pid>/maps` 把一個程序對應的進程的內存映象看個大概。

```
$ cat /proc/self/maps   #查看cat程序自身加載後對應進程的內存映像
08048000-0804c000 r-xp 00000000 03:01 273716     /bin/cat
0804c000-0804d000 rw-p 00003000 03:01 273716     /bin/cat
0804d000-0806e000 rw-p 0804d000 00:00 0          [heap]
b7c46000-b7e46000 r--p 00000000 03:01 87528      /usr/lib/locale/locale-archive
b7e46000-b7e47000 rw-p b7e46000 00:00 0
b7e47000-b7f83000 r-xp 00000000 03:01 466875     /lib/libc-2.5.so
b7f83000-b7f84000 r--p 0013c000 03:01 466875     /lib/libc-2.5.so
b7f84000-b7f86000 rw-p 0013d000 03:01 466875     /lib/libc-2.5.so
b7f86000-b7f8a000 rw-p b7f86000 00:00 0
b7fa1000-b7fbc000 r-xp 00000000 03:01 402817     /lib/ld-2.5.so
b7fbc000-b7fbe000 rw-p 0001b000 03:01 402817     /lib/ld-2.5.so
bfcdf000-bfcf4000 rw-p bfcdf000 00:00 0          [stack]
ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
```

關於程序加載和進程內存映像的更多細節請參考[《C 語言程序緩衝區注入分析》][100]。

[100]: 02-chapter5.markdown

到這裡，關於命令行的祕密都被“曝光”了，可以開始寫自己的命令行解釋程序了。

關於進程的相關操作請參考[《進程與進程的基本操作》][101]。

[101]: 02-chapter7.markdown

補充：上面沒有討論到一個比較重要的內容，那就是即使 `execve` 找到了某個可執行文件，如果該文件屬主沒有運行該程序的權限，那麼也沒有辦法運行程序。可通過 `ls -l` 查看程序的權限，通過 `chmod` 添加或者去掉可執行權限。

文件屬主具有可執行權限時才可以執行某個程序：

```
$ whoami
falcon
$ ls -l hello  #查看用戶權限(第一個x表示屬主對該程序具有可執行權限
-rwxr-xr-x 1 falcon users 6383 2000-01-23 07:59 hello*
$ ./hello
Hello World
$ chmod -x hello  #去掉屬主的可執行權限
$ ls -l hello
-rw-r--r-- 1 falcon users 6383 2000-01-23 07:59 hello
$ ./hello
-bash: ./hello: Permission denied
```

<span id="toc_16100_6031_13"></span>
## 參考資料

- Linux 啟動過程：`man boot-scripts`
- Linux 內核啟動參數：`man bootparam`
- `man 5 passwd`
- `man shadow`
- 《UNIX 環境高級編程》，進程關係一章
