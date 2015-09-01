# 進程的內存映像

-    [前言](#toc_14856_6356_1)
-    [進程內存映像表](#toc_14856_6356_2)
-    [在程序內部打印內存分佈信息](#toc_14856_6356_3)
-    [在程序內部獲取完整內存分佈信息](#toc_14856_6356_4)
-    [後記](#toc_14856_6356_5)
-    [參考資料](#toc_14856_6356_6)


<span id="toc_14856_6356_1"></span>
## 前言

在閱讀《UNIX 環境高級編程 》的第 14 章時，看到一個“打印不同類型的數據所存放的位置”的例子，它非常清晰地從程序內部反應了“進程的內存映像”，通過結合它與[《Gcc 編譯的背後》][1]和[《緩衝區溢出與注入分析》][2]的相關內容，可以更好地輔助理解相關的內容。

[1]: 02-chapter2.markdown
[2]: 02-chapter5.markdown

<span id="toc_14856_6356_2"></span>
## 進程內存映像表

首先回顧一下[《緩衝區溢出與注入》][2]中提到的"進程內存映像表"，並把共享內存的大概位置加入該表：

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

<span id="toc_14856_6356_3"></span>
## 在程序內部打印內存分佈信息

為了能夠反應上述內存分佈情況，這裡在《UNIX 環境高級編程 》的程序 14-11 的基礎上，添加了一個已經初始化的全局變量（存放在已經初始化的數據段內），並打印了它以及 `main` 函數(處在代碼正文部分)的位置。

```
/**
 * showmemory.c -- print the position of different types of data in a program in the memory
 */

#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <stdlib.h>

#define ARRAY_SIZE 4000
#define MALLOC_SIZE 100000
#define SHM_SIZE 100000
#define SHM_MODE (SHM_R | SHM_W)    /* user read/write */

int init_global_variable = 5;    /* initialized global variable */
char array[ARRAY_SIZE];        /* uninitialized data = bss */

int main(void)
{
    int shmid;
    char *ptr, *shmptr;

    printf("main: the address of the main function is %x\n", main);
    printf("data: data segment is from %x\n", &init_global_variable);
    printf("bss: array[] from %x to %x\n", &array[0], &array[ARRAY_SIZE]);
    printf("stack: around %x\n", &shmid);   
   
    /* shmid is a local variable, which is stored in the stack, hence, you
     * can get the address of the stack via it*/

    if ( (ptr = malloc(MALLOC_SIZE)) == NULL) {
        printf("malloc error!\n");
        exit(-1);
    }
    printf("heap: malloced from %x to %x\n", ptr, ptr+MALLOC_SIZE);

    if ( (shmid = shmget(IPC_PRIVATE, SHM_SIZE, SHM_MODE)) < 0) {
        printf("shmget error!\n");
        exit(-1);
    }

    if ( (shmptr = shmat(shmid, 0, 0)) == (void *) -1) {
        printf("shmat error!\n");
        exit(-1);
    }
    printf("shared memory: attached from %x to %x\n", shmptr, shmptr+SHM_SIZE);

    if (shmctl(shmid, IPC_RMID, 0) < 0) {
        printf("shmctl error!\n");
        exit(-1);
    }

    exit(0);
}
```

該程序的運行結果如下：

```
$  make showmemory
cc     showmemory.c   -o showmemory
$ ./showmemory
main: the address of the main function is 804846c
data: data segment is from 80498e8
bss: array[] from 8049920 to 804a8c0
stack: around bfe3e224
heap: malloced from 804b008 to 80636a8
shared memory: attached from b7da7000 to b7dbf6a0
```

上述運行結果反應了幾個重要部分數據的大概分佈情況，比如 `data` 段（那個初始化過的全局變量就位於這裡）、bss 段、stack、heap，以及 shared memory 和main（代碼段）的內存分佈情況。

<span id="toc_14856_6356_4"></span>
## 在程序內部獲取完整內存分佈信息

不過，這些結果還是沒有精確和完整地反應所有相關信息，如果要想在程序內完整反應這些信息，結合[《Gcc編譯的背後》][1]，就不難想到，我們還可以通過擴展一些已經鏈接到可執行文件中的外部符號來獲取它們。這些外部符號全部定義在可執行文件的符號表中，可以通過 `nm/readelf -s/objdump -t` 等查看到，例如：

```
$ nm showmemory
080497e4 d _DYNAMIC
080498b0 d _GLOBAL_OFFSET_TABLE_
080486c8 R _IO_stdin_used
         w _Jv_RegisterClasses
080497d4 d __CTOR_END__
080497d0 d __CTOR_LIST__
080497dc d __DTOR_END__
080497d8 d __DTOR_LIST__
080487cc r __FRAME_END__
080497e0 d __JCR_END__
080497e0 d __JCR_LIST__
080498ec A __bss_start
080498dc D __data_start
08048680 t __do_global_ctors_aux
08048414 t __do_global_dtors_aux
080498e0 D __dso_handle
         w __gmon_start__
0804867a T __i686.get_pc_thunk.bx
080497d0 d __init_array_end
080497d0 d __init_array_start
08048610 T __libc_csu_fini
08048620 T __libc_csu_init
         U __libc_start_main@@GLIBC_2.0
080498ec A _edata
0804a8c0 A _end
080486a8 T _fini
080486c4 R _fp_hw
08048328 T _init
080483f0 T _start
08049920 B array
08049900 b completed.1
080498dc W data_start
         U exit@@GLIBC_2.0
08048444 t frame_dummy
080498e8 D init_global_variable
0804846c T main
         U malloc@@GLIBC_2.0
080498e4 d p.0
         U printf@@GLIBC_2.0
         U shmat@@GLIBC_2.0
         U shmctl@@GLIBC_2.2
         U shmget@@GLIBC_2.0
```

第三列的符號在我們的程序中被擴展以後就可以直接引用，這些符號基本上就已經完整地覆蓋了相關的信息了，這樣就可以得到一個更完整的程序，從而完全反應上面提到的內存分佈表的信息。

```
/**
 * showmemory.c -- print the position of different types of data in a program in the memory
 */

#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <stdlib.h>

#define ARRAY_SIZE 4000
#define MALLOC_SIZE 100000
#define SHM_SIZE 100000
#define SHM_MODE (SHM_R | SHM_W)        /* user read/write */

                                        /* declare the address relative variables */
extern char _start, __data_start, __bss_start, etext, edata, end;
extern char **environ;

char array[ARRAY_SIZE];         /* uninitialized data = bss */

int main(int argc, char *argv[])
{
    int shmid;
    char *ptr, *shmptr;

    printf("===== memory map =====\n");
    printf(".text:\t0x%x->0x%x (_start, code text)\n", &_start, &etext);
    printf(".data:\t0x%x->0x%x (__data_start, initilized data)\n", &__data_start, &edata);
    printf(".bss: \t0x%x->0x%x (__bss_start, uninitilized data)\n", &__bss_start, &end);

    /* shmid is a local variable, which is stored in the stack, hence, you
     * can get the address of the stack via it*/

    if ( (ptr = malloc(MALLOC_SIZE)) == NULL) {
        printf("malloc error!\n");
        exit(-1);
    }

    printf("heap: \t0x%x->0x%x (address of the malloc space)\n", ptr, ptr+MALLOC_SIZE);

    if ( (shmid = shmget(IPC_PRIVATE, SHM_SIZE, SHM_MODE)) < 0) {
        printf("shmget error!\n");
        exit(-1);
    }

    if ( (shmptr = shmat(shmid, 0, 0)) == (void *) -1) {
        printf("shmat error!\n");
        exit(-1);
    }
    printf("shm  :\t0x%x->0x%x (address of shared memory)\n", shmptr, shmptr+SHM_SIZE);

    if (shmctl(shmid, IPC_RMID, 0) < 0) {
        printf("shmctl error!\n");
        exit(-1);
    }

    printf("stack:\t <--0x%x--> (address of local variables)\n", &shmid);   
    printf("arg:  \t0x%x (address of arguments)\n", argv);
    printf("env:  \t0x%x (address of environment variables)\n", environ);

    exit(0);
}
```

運行結果：

```
$ make showmemory
$ ./showmemory
===== memory map =====
.text:    0x8048440->0x8048754 (_start, code text)
.data:    0x8049a3c->0x8049a48 (__data_start, initilized data)
.bss:     0x8049a48->0x804aa20 (__bss_start, uninitilized data)
heap:     0x804b008->0x80636a8 (address of the malloc space)
shm  :    0xb7db6000->0xb7dce6a0 (address of shared memory)
stack:     <--0xbff85b64--> (address of local variables)
arg:      0xbff85bf4 (address of arguments)
env:      0xbff85bfc (address of environment variables)
```

<span id="toc_14856_6356_5"></span>
## 後記

上述程序完整地勾勒出了進程的內存分佈的各個重要部分，這樣就可以從程序內部獲取跟程序相關的所有數據了，一個非常典型的例子是，在程序運行的過程中檢查代碼正文部分是否被惡意篡改。

如果想更深地理解相關內容，那麼可以試著利用 `readelf`，`objdump` 等來分析 ELF 可執行文件格式的結構，並利用 `gdb` 來了解程序運行過程中的內存變化情況。

<span id="toc_14856_6356_6"></span>
## 參考資料

- [Gcc 編譯的背後（第二部分：彙編和鏈接）][1]
- [緩衝區溢出與注入分析][2]
- 《Unix 環境高級編程》第 14 章，程序 14-11
