# 前言

-    [背景](#toc_1235_30984_1)
-    [現狀](#toc_1235_30984_2)
-    [計劃](#toc_1235_30984_3)


<span id="toc_1235_30984_1"></span>
## 背景

2007 年開始系統地學習 Shell 編程，並在[蘭大開源社區](http://oss.lzu.edu.cn)寫了序列文章。

在編寫[《Shell 編程範例》](http://tinylab.gitbooks.io/shellbook)文章的[《進程操作》](http://tinylab.gitbooks.io/shellbook/content/zh/chapters/01-chapter7.html)一章時，為了全面瞭解進程的來龍去脈，對程序開發過程的細節、ELF 格式的分析、進程的內存映像等進行了全面地梳理，後來搞得“雪球越滾越大”，甚至脫離了 Shell 編程關注的內容。所以想了個小辦法，“大事化小，小事化了”，把涉及到的內容進行了分解，進而演化成另外一個完整的序列。

2008 年 3 月 1 日，當初步完成整個序列時，做了如下的小結：

> 到今天，關於"Linux 下 C 語言開發過程"的一個簡單視圖總算粗略地完成了，從寒假之前的一段時間到現在過了將近一個月左右吧。寫這個主題的目的源自“Shell 編程範例之進程操作”，當寫到這一章時，突然對進程的由來、本身和去向感到“迷惑不解”。所以想著好好花些時間來弄清楚它們，現在發現，這個由來就是這裡的程序開發過程，進程來自一個普通的文本文件，在這裡是 C 語言程序，C 語言程序經過編輯、預處理、編譯、彙編、鏈接、執行而成為一個進程；而進程本身呢？當一個可執行文件被執行以後，有了 exec 調用，被程序解釋器映射到了內存中，有了它的內存映像；而進程的去向呢？通過不斷地執行指令和內存映像的變化，進程完成著各項任務，等任務完成以後就可以退出了（exit）。
>
> 這樣一份視圖實際上是在寒假之前繪好的，可以從下圖中看到它；不過到現在才明白背後的很多細節。這些細節就是這個序列的每個篇章，可以對照“視圖”來閱讀它們。

![C語言程序開發過程視圖](pic/c_dev_procedure.jpg)

<span id="toc_1235_30984_2"></span>
## 現狀

目前整個序列大部分都已經以 Blog 的形式寫完，大體結構目下：

-   [《把 VIM 打造成源代碼編輯器》][1]
    -   源代碼編輯過程：用 VIM 編輯代碼的一些技巧
    -   更新時間：2008-2-22


-   [《GCC 編譯的背後》][2]
    -   編譯過程：預處理、編譯、彙編、鏈接
    -   第一部分：《預處理和編譯》（更新時間：2008-2-22）
    -   第二部分：《彙編和鏈接》（更新時間：2008-2-22）


-   [《程序執行的那一剎那 》][3]
    -   執行過程：當從命令行輸入一個命令之後
    -   更新時間：2008-2-15


-   [《進程的內存映像》][4] 
    -   進程加載過程：程序在內存裡是個什麼樣子？
    -   第一部分（討論“緩衝區溢出和注入”問題）（更新時間：2008-2-13）
    -   第二部分（討論進程的內存分佈情況）（更新時間：2008-6-1）


-   [《進程和進程的基本操作》][5]
    -   進程操作：描述進程相關概念和基本操作
    -   更新時間：2008-2-21


-   [《動態符號鏈接的細節》][6]
    -   動態鏈接過程：函數 puts/printf 的地址在哪裡？
    -   更新時間：2008-2-26


-   [《打造史上最小可執行ELF文件》][7]
    -   ELF 詳解：從”減肥”的角度一層一層剖開 ELF 文件，最終獲得一個可打印 Hello World 的 **45** 字節 ELF 可執行文件
    -   更新時間：2008-2-23


-   [《代碼測試、調試與優化小結》][8]
    -   程序開發過後：內存溢出了嗎？有緩衝區溢出？代碼覆蓋率如何測試呢？怎麼調試彙編代碼？有哪些代碼優化技巧和方法呢？
    -   更新時間：2008-2-29

[1]: http://www.tinylab.org/make-vim-source-code-editor/
[2]: http://www.tinylab.org/behind-the-gcc-compiler/
[3]: http://www.tinylab.org/program-execution-the-moment/
[4]: http://www.tinylab.org/process-memory-image/ 
[5]: http://www.tinylab.org/process-and-basic-operation/
[6]: http://www.tinylab.org/details-of-a-dynamic-symlink/
[7]: http://www.tinylab.org/as-an-executable-file-to-slim-down/
[8]: http://www.tinylab.org/testing-debugging-and-optimization-of-code-summary/

<span id="toc_1235_30984_3"></span>
## 計劃

考慮到整個 Linux 世界的蓬勃發展，Linux 和 C 語言的應用環境越來越多，相關使用群體會不斷增加，所以最近計劃把該序列重新整理，以自由書籍的方式不斷更新，以便惠及更多的讀者。

打算重新規劃、增補整個序列，並以開源項目的方式持續維護，並通過 [泰曉科技|TinLab.org](http://tinylab.org) 平臺接受讀者的反饋，直到正式發行出版。

自由書籍將會維護在 [泰曉科技](http://tinylab.org) 的[項目倉庫](https://github.com/tinyclub/open-c-book)中。項目相關信息如下：

-   項目首頁：<http://www.tinylab.org/project/hello-c-world/>
-   代碼倉庫：[https://github.com/tinyclub/open-c-book.git](https://github.com/tinyclub/open-c-book)

歡迎大家指出本書初稿中的不足，甚至參與到相關章節的寫作、校訂和完善中來。

如果有時間和興趣，歡迎參與。可以通過 [泰曉科技](http://www.tinylab.org/about/) 聯繫我們，或者直接關注微博[@泰曉科技](http://weibo.com/tinylaborg)並私信我們。
