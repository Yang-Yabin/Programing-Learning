#### 存储映射I/O

> wanqiu：**:nail_care:内存映射是将磁盘或内存内核区的文件或设备文件映射到内存用户区，便于应用程序进行直接读写操作。**

![](imgs/进程间通信的三种方式.jpg)

​                                                                                     Unix进程间共享信息的三种方式（《Unix网络编程：卷二》）

> 存储映射 I/O（memory-mapped I/O）是一种基于内存区域的高级 I/O 操作，**它能将一个文件映射到进程地址空间中的一块内存区域中，当从这段内存中读数据时，就相当于读文件中的数据（对文件进行 read 操 作），将数据写入这段内存时，则相当于将数据直接写入文件中（对文件进行 write 操作）。**这样就可以在 不使用基本 I/O 操作函数 read()和 write()的情况下执行 I/O 操作。

##### `mmap()`和 `munmap()`函数

###### `mmap()`函数

为了实现存储映射 I/O 这一功能，我们需要告诉内核将一个给定的文件映射到进程地址空间（用户区）中的一块内存区域中，这由系统调用 `mmap()`来实现。

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

`addr：`参数 `addr` 用于指定映射到内存区域的起始地址。通常将其设置为 NULL，这表示由系统选择该 映射区的起始地址，这是最常见的设置方式；如果参数 `addr` 不为 NULL，则表示由自己指定映射区的起始 地址，此函数的返回值是该映射区的起始地址。

length：参数 length 指定映射长度，表示将文件中的多大部分映射到内存区域中，以字节为单位，譬如 length=1024 * 4，表示将文件的 4K 字节大小映射到内存区域中。

`offset：`文件映射的偏移量，通常将其设置为 0，表示从文件头部开始映射；所以参数 offset 和参数 length 就确定了文件的起始位置和长度，将文件的这部分映射到内存区域中。

`fd：`文件描述符，指定要映射到内存区域中的文件。

`prot：`参数 `prot` 指定了映射区的保护要求：

`PROT_EXEC：`映射区可执行； 

`PROT_READ：`映射区可读； 

`PROT_WRITE：`映射区可写；

`PROT_NONE：`映射区不可访问。

`flags：`参数 flags 可影响映射区的多种属性：在众多标志当 中，通常情况下，参数 flags 中只指定了 MAP_SHARED

`MAP_SHARED：`此标志指定当对映射区写入数据时，数据会写入到文件中，也就是会将写入到映 射区中的数据更新到文件中，并且允许其它进程共享。

返回值：成功情况下，函数的返回值便是**映射区的起始地址**。

###### `munmap()`解除映射

> 通过 open()打开文件，需要使用 close()将将其关闭；同理，**通过 `mmap()`将文件映射到进程地址空间中 的一块内存区域中，当不再需要时，必须解除映射，使用 `munmap()`解除映射关系。**

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
```

munmap()系统调用解除指定地址范围内的映射，参数 addr 指定待解除映射地址范围的起始地址，它必 须是系统页大小的整数倍；参数 length 是一个非负整数，指定了待解除映射区域的大小（字节数）

当进程终止时也会自动解除映射（如果程序中没有显式调用 munmap()），但调用 close() 关闭文件时并不会解除映射。

> **通常将参数 addr 设置为 mmap()函数的返回值，将参数 length 设置为 mmap()函数的参数 length，表示解 除整个由 mmap()函数所创建的映射。**

##### 普通 I/O 与存储映射 I/O 

###### 普通 I/O 的缺点

> 普通 I/O 方式一般是通过调用 read()和 write()函数来实现对文件的读写，使用 read()和 write()读写文件 时，函数经过层层的调用后，才能够最终操作到文件，中间涉及到很多的函数调用过程，数据需要在不同的 缓存间倒腾，效率会比较低。同样使用标准 I/O（库函数 fread()、fwrite()）也是如此，本身标准 I/O 就是对 普通 I/O 的一种封装。 那既然效率较低，为啥还要使用这种方式呢？原因在于，只有当数据量比较大时，效率的影响才会比较 明显，如果数据量比较小，影响并不大，使用普通的 I/O 方式还是非常方便的。

###### 存储映射I/O 的优点

存储映射 I/O 的实质其实是共享，与 IPC 之内存共享很相似。譬如执行一个文件复制操作来说，对于普 通 I/O 方式，首先需要将源文件中的数据读取出来存放在一个应用层缓冲区中，接着再将缓冲区中的数据写 入到目标文件中。

而对于存储映射 I/O 来说，由于源文件和目标文件都已映射到了应用层的内存区域中，所以直接操作映射区来实现文件复制。

应用层与内核层是不能直接进行交互的，必须要通过操作系统提供的系统调用或库函数来与内核进行数据交互，包括操作硬件。通过存储映射 I/O 将文件直接映射到应用程序地址空间中的一块内存区域中，也就是映射区；直接 将磁盘文件直接与映射区关联起来，不用调用 read()、write()系统调用，直接对映射区进行读写操作即可操 作磁盘上的文件，而磁盘文件中的数据也可反应到映射区中，这就是一种共享，可以认为映射区就是应用层 与内核层之间的共享内存。

**使用存储映射 I/O 在 进行大数据量操作时比较有效；对于少量数据，使用普通 I/O 方式更加方便。**

