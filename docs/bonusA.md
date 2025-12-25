# Bonus A：FAT32 文件系统

!!! danger "DDL"

    - 代码、报告：2026-01-12 23:59
    - 验收：无

## 实验简介

在此前的实验中，我们的操作系统虽然已经支持多进程与用户态程序，但所有程序与数据都只存在于内存中：用户态 ELF 被静态打包进内核镜像，系统无法读写真实磁盘，也谈不上持久化存储。

在本次实验中，我们将实现一个简单的 FAT32 文件系统驱动，使得内核能够读写文件系统镜像中的数据，并实现从文件系统中加载并运行用户态程序。

为了实现这一目标，我们需要考虑以下几个方面的问题：

1. **内核如何统一管理“文件”？** 为了让标准输入输出、磁盘文件等对象能够被统一访问，我们需要：
    - 设计一种**文件抽象**来描述各种类型的文件对象；
    - 实现统一的文件操作接口。
2. **内核如何与磁盘进行交互，并读写 FAT32 文件系统？** 我们需要：
    - 引入 VirtIO 块设备的驱动程序作为磁盘读写的底层基础；
    - 在其上根据 FAT32 文件系统的磁盘布局实现文件的打开、读取与写入等功能。
3. **内核如何从文件系统中加载用户态程序？** 我们需要：
    - 完善进程的生命周期管理，通过 `fork` + `execve` 的方式创建用户态进程；
    - 在加载 ELF 文件和发生缺页异常时从文件系统中读取数据；
    - 在进程退出后，由父进程通过 `wait` 来正确回收它。

## Part 0：合并代码

### 更新代码

现在你位于 `lab5` 分支。你需要创建 `bonus-a` 分支，合并上游的代码：

```shell
git checkout -b bonus-a
git fetch upstream
git merge upstream/bonus-a
```

### Qemu 加载文件系统

在 Lab0 中，我们已经尝试过用 qemu 运行自己编译的 Linux 内核镜像，我们同时挂载了一个 `rootfs.ext2` 的文件系统镜像作为根文件系统。在本次实验中，我们将把它替换为我们预先准备好的 `disk.img` 文件系统镜像，它包含了一个 FAT32 分区。

你可以在 `Makefile` 中看到如下的挂载命令：

```Makefile
-global virtio-mmio.force-legacy=false \
-drive file=$(FAT32_PATH),format=raw,id=hd0,if=none \
-device virtio-blk-device,drive=hd0 \
```

请你解压 `disk.img.zip`，并将解压得到的 `disk.img` 文件放置在项目根目录下准备挂载。

### VirtIO 块设备驱动

VirtIO 是一个开放标准，它定义了一种协议，用于不同类型的驱动程序和设备之间的通信。在 QEMU 上我们可以基于 VirtIO 使用许多模拟出来的外部设备。本次实验中，我们使用 VirtIO 模拟存储设备，并在其上构建文件系统。

VirtIO 块设备驱动的代码已经位于 `kernel/fs/virtio.c` 中，无需大家实现。你可以**直接使用其中提供的 `virtio_blk_read_sector()` 和 `virtio_blk_write_sector()` 函数来对磁盘扇区进行读写操作**。

!!! note "VirtIO 交互方式"
    如果你对 VirtIO 设备的交互方式感兴趣，欢迎阅读 [virtio_blk块设备驱动程序](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter9/2device-driver-3.html) 深入了解。在本次实验中，我们采用简单的阻塞轮询方式与 VirtIO 设备交互，另外还有基于中断的交互方式可以实现更高效的设备驱动。

为了初始化 VirtIO 并实现 MBR 引导 (后续会介绍)，我们还需要完成如下的准备工作：

- 在 `kernel/arch/riscv/kernel/head.S` 中的 `task_init` 函数之后调用 `virtio_dev_init()` 和 `mbr_init()`，完成 VirtIO 设备和 MBR 的初始化；
- 在 `kernel/arch/riscv/kernel/vm.c` 中的 `setup_vm_final()` 函数中添加对 VritIO 外设的映射。此外，你可能还需要修改 `copy_pgd()` 和 `free_pgd()` 函数，把这一部分的映射也作为内核映射的一部分进行处理，无需在每个进程页表中重复映射。

```c title="kernel/arch/riscv/kernel/vm.c"
// map device memory -|W|R|V
create_mapping(swapper_pg_dir, (void *)io_to_virt(VIRTIO_START),
            (void *)VIRTIO_START, VIRTIO_SIZE * VIRTIO_COUNT,
            PTE_W | PTE_R);
```

### NISH

我们为大家提供了用户态程序 "nish" (Not Implemented SHell) 来与我们在实验中完成的 kernel 进行交互。它提供了简单的用户交互和文件读写功能，有如下的命令：

```shell
echo [string]   # 将 string 输出到 stdout
cat [path]      # 将路径为 path 的文件的内容输出到 stdout
edit [path] [offset] [string] # 将路径为 path 的文件，偏移量为 offset 的部分开始，写为 string
```

请大家阅读 `kernel/user/src/main.c` 中 `nish` 的实现，理解它是如何通过系统调用与内核进行交互的。在完成本次实验中，你可以使用 `nish` 来测试你实现的文件系统功能。

### 父子进程与 wait 机制

我们在 `kernel/arch/riscv/include/proc.h` 中为进程结构体 `struct task_struct` 添加了如下字段：

```c
struct task_struct {
    /* ... */
    struct task_struct *parent; /**< 父进程 */
    struct completion child_exit_comp; /**< 子进程退出的 completion */
    int exit_code; /**< 退出码 */
};
```

如果你不完成这一部分的实现，也不会影响本次实验的 FAT32 文件系统功能。但是，**你可能需要在 `task_init()` 时初始化 `child_exit_comp`，以避免后续实验中出现未初始化的错误。**

## Part 1：设计文件抽象

### 在 C 中实现继承与多态

在 Unix 哲学中有着 **“Everything is a File”** 的设计思想。同学们应该都已经学习过《面向对象程序设计》课程，现在让我们用 OOP 来理解文件抽象这一思想。

计算机系统中的一切事物，都可以**视为一串字节**，可以进行**读（read）、写（write）、寻址（lseek）**等基本操作，比如：

- 文件系统中的文件、一个磁盘、一块内存区域：它们都是一串字节。
- 一个外设，比如显卡：也可以视为一串字节。这里作一个不太准确的描述：把需要显示的图像写入到显卡的显存中的特定区域，显卡就能将其输出为数字信号，显示到屏幕上。

对于有一套公共接口的资源，我们自然会想为其创建一个抽象**基类**。我们创建的基类叫做“文件”，它定义在 `kernel/include/fs.h` 中，包含了文件描述符、权限、当前偏移等通用属性：

```c title="kernel/include/fs.h"
struct file {
    struct list_head list;
    int fd;     /* 文件描述符编号 */
    uint32_t opened;  /* 文件是否打开 */
    uint32_t perms;   /* 文件权限 */
    int64_t cfo;   /* 当前文件偏移 */

    /* 文件操作函数指针 */
    int64_t (*lseek)(struct file *file, int64_t offset, uint64_t whence);
    int64_t (*write)(struct file *file, const void *buf, uint64_t len);
    int64_t (*read)(struct file *file, void *buf, uint64_t len);

    char path[MAX_PATH_LENGTH]; /* 文件路径 */
};
```

显然，读写内存和读写磁盘的具体实现应该不同。在 C++ 中，接下来我们会创建继承自该抽象“文件”的子类，比如文件系统中的文件、磁盘、内存区域等单独创建子类，为不同类型的文件实现其具体的读写方法。但操作系统内核并不是用 C++ 写的，应该怎么实现类似的功能呢？

答案是：

- **使用函数指针和联合体（union）**分别放置具体实现的函数和数据
- **显式标注**子类型，相关代码需根据类型执行不同的操作

```c title="kernel/include/fs.h"
struct file {
    uint32_t fs_type;  /* 文件系统类型 */
    union {
        struct fat32_file fat32_file;
    };
}
```

例如，创建 FAT32 文件系统的文件时，我们将 `fs_type` 设置为 `FS_TYPE_FAT32`，并将 `lseek()`、`write()` 和 `read()` 这几个函数指针指向 FAT32 文件系统的实现，比如 `fat32_write()` 等。相关代码需识别 `fs_type`，并使用 union 中的 `fat32_file` 成员来存取 FAT32 文件系统中的具体信息。

通过上述的设计，我们为所有“文件”创建了统一的抽象接口，却又能访问到不同的文件系统。用 OOP 的术语说，我们实现了部分的**继承、多态功能**。

### 进程与文件描述符

接下来，我们把 `struct file` 绑定到进程结构体 `struct task_struct` 中：

```c
struct files_struct {
 struct list_head fd_list;
};

struct task_struct {
    /* ... */
 struct files_struct files; /**< 进程的文件描述符表 */
};
```

这样一来，每个进程都可以通过该链表打开和管理多个文件。所谓**文件描述符（file descriptor）就是一个数字**，用于在进程的文件描述符表中寻找对应的文件对象。当前进程总是可以通过 `current->files` 访问自己的文件并进行文件操作。

有几个 FD 已经在习惯上被划定用于特定功能：

- 标准输入 (`fd = 0`)
- 标准输出 (`fd = 1`)
- 标准错误输出 (`fd = 2`)

### Task 1：实现标准输入输出

在之前的实验中，我们已经实现了用户态的 `write` 系统调用，但当时：

- 我们并没有实现文件抽象；
- 默认仅支持 `fd = 1` 的标准输出，将来自用户态的数据输出到控制台。

现在，我们需要以文件抽象为基础，完善 `write` 系统调用的实现。具体来说：

- 标准输出 (`fd = 1`) 应当被视为一个特殊的文件对象，它实现了一个 `write()` 函数，能够将数据输出到控制台；
- `write()` 系统调用应当通过当前进程的文件描述符表 (`current->files`) 找到 `fd = 1` 的文件对象，并调用它的 `write()` 函数 `stdout_write()` 完成数据输出。

为了完成 `write` 系统调用的改造并实现标准输入输出，你需要完成以下工作：

1. **管理文件对象的生命周期。**
    - **请你实现** `kernel/fs/fs.c` 中的一系列函数，完成文件对象的初始化、创建、拷贝、释放等功能，并在 `kernel/arch/riscv/kernel/proc.c` 中正确地调用它们，确保每个进程的文件描述符表正确地初始化、拷贝和释放。
2. **创建标准输入输出对象。**
    - **请你实现** `kernel/fs/fs.c` 中的 `file_init` 函数，为每个进程初始化时都创建默认的 stdio 文件对象，并将它们添加到进程的文件描述符表中。其中，我们对 fd 编号做出如下约定：

    ``` title="标准文件描述符编号"
    stdin (0)
    stdout (1)
    stderr (2)
    ```

3. **实现标准输入输出函数。**
    - **请你阅读** `kernel/fs/vfs.c` 中的 `uart_getchar()` 和 `sbi_puts()` 函数，理解它们如何与 UART 设备交互完成字符的输入输出。
    - **请你实现** `kernel/fs/vfs.c` 中的 `stdout_write()`, `stderr_write()` 和 `stdin_read()` 函数，分别完成标准输出、标准错误输出和标准输入的功能。
    !!! tip "内核态与用户态地址空间"
        注意到 `write` 和 `read` 系统调用的缓冲区参数 `buf` 是用户态地址空间的地址，而内核态无法直接访问用户态地址空间。你可以参考 `lab5` 中实现的 `copy_from_user()` 函数，**以合适的方式实现内核态和用户态地址空间的数据拷贝**。
4. **改造 `write` 和 `read` 系统调用。**
    - **请你实现** `kernel/arch/riscv/kernel/syscall.c` 中的 `sys_write` 和 `sys_read` 函数，完成对文件描述符的查找，并调用对应文件对象的 `write` 和 `read` 函数。

!!! success "完成条件"
    在完成上述任务后，你的系统应当能够正确地处理用户态程序对标准输入输出的读写操作。

    我们的用户态程序在启动时会调用 `write` 系统调用测试 `stdout` 和 `stderr` 的功能，你可以在控制台观察到如下输出：
    ```text
    hello, stdout!
    hello, stderr!
    ```
    接下来，我们的用户态程序会**通过 `read` 系统调用从标准输入读取数据，并将其回显到标准输出**。你可以在控制台输入任意内容，观察到相同的内容被输出。此外， `echo` 命令应当能够正常工作：
    ```shell
    SHELL > echo "test"
    test
    ```

## Part 2：FAT32 文件系统

### 背景介绍

**MBR (主引导扇区)** 是计算机开机后访问硬盘时必须要读取的首个扇区，它位于硬盘的第 0 个扇区。在本次实验中，我们只会用到其中 16 字节的**磁盘分区表（DPT）**，解析其中的内容，并找到 FAT 分区的起始与结束相关信息。

**文件分配表 (File Allocation Table，FAT)** 是一种由微软发明并拥有部分专利的文件系统。最早的 FAT 文件系统直接使用扇区号来作为存储的索引，但是这样做的缺点是显然易见的：当磁盘的大小不断扩大，存储扇区号的位数越来越多，越发占用存储空间。以 32 位扇区号的文件系统为例，如果磁盘的扇区大小为 512B，那么文件系统能支持的最大磁盘大小仅为 2TB。因此，FAT32 引入了新的存储管理单位：**簇**。一个簇包含一个或多个扇区，文件系统中记录的索引不再为扇区号，而是簇号，以此来支持更大的存储设备。

在本次实验中我们仅需实现 FAT32 文件系统中很小一部分功能，我们为实验中的测试做如下限制：

- 文件名长度小于等于 8 个字符，并且不包含后缀名和字符 `.` .
- 不包含目录的实现，所有文件都保存在磁盘根目录 `/fat32/` 下。
- 不涉及磁盘上文件的创建和删除。
- 不涉及文件大小的修改。

!!! note "FAT32 参考资料"
    **以下内容对完成本次实验非常重要，请你仔细阅读。**

    - [FAT32文件系统？盘它！](https://www.youtube.com/watch?v=YfDU6g0CmZE&t=344s) 里提到了“块”的概念，可以对标本次实验里的“扇区”（sector）
        - `fat32_init` 函数里各种参数的计算方式可以参考本视频；
    - [Microsoft FAT Specification](https://academy.cba.mit.edu/classes/networking_communications/SD/FAT.pdf) 是本次实验参考的标准，关于 `fat32_bpb` `fat32_dir_entry` 等结构体各个字段的含义，可以直接参考本材料，例如：
        - Page 7 讲解了 `fat32_bpb`，在 `fat32_init` 中会使用到；
        - Page 23 处讲解了 `fat32_dir_entry`，在文件操作函数中会使用到。

### FAT32 分区初始化

FAT32 分区的第一个扇区中存储了关于这个分区的元数据，我们需要实现 `fat32_init()` 函数，读取并解析这些元数据，为后续的文件系统操作提供支持。我们在 `kernel/include/fat32.h` 提供了 `fat32_bpb` 和 `fat32_volume` 这两个数据结构的定义，其中：

- `fat32_bpb`（FAT32 BIOS Parameter Block）是一个物理扇区，其中对应的是这个分区的元数据。我们首先需要将该扇区的内容读到一个 `fat32_bpb` 数据结构中进行解析。
- `fat32_volume` 需要根据 `fat32_bpb` 中的数据来进行计算并完成初始化，它存储了后续代码中我们需要用到的元数据。

!!! note "FAT32 中扇区的排列方式与用途"
    FAT32 中的扇区以如下方式按顺序排列：

    - 第一个扇区即为 BPB，在这之后紧跟着的是一些保留扇区；
    - 接下来是第一个 FAT 表所在的扇区（这个表会占用多个扇区，需要从 BPB 中读取），和第二个 FAT 表所在的扇区（为第一个 FAT 表的备份）；
    - 再之后就是 FAT 数据区。

!!! tip "关于簇和扇区"

    为了简单起见，本次实验中每个簇都只包含 1 个扇区，所以你的代码可能会以各种灵车的方式跑起来，但是你**仍需要对簇和扇区有所区分**，并在报告里有所体现。

### FAT32 文件操作：打开、读取与写入

**打开文件**。我们在 `fat32_open_file()` 函数中实现打开文件的功能。我们需要读取出被打开的文件所在的簇和目录项位置的信息，来供后面 `read`, `write`, `lseek` 使用。具体来说，我们需要：

- 遍历数据区开头的根目录扇区；
- 找到 `name` 和 `path` 末尾的 `filename` 相匹配的 `fat32_dir_entry` 目录项结构体；
- 从中得到 FAT32 文件和目录项的相关信息。

其中，文件和目录项的信息存储在 `kernel/include/fs.h` 中定义的如下两个结构体中：

```c
struct fat32_dir {
 uint32_t cluster;   // 文件的目录项所在的簇
 uint32_t index;     // 文件的目录项是该簇中的第几个目录项
};

struct fat32_file {
 uint32_t cluster;       // 文件开头所在的簇
 struct fat32_dir dir;   // 文件的目录项信息
};
```

!!! tip "为什么我匹配不到要打开的文件"
    如果你使用 `memcmp` 来逐一比较 FAT32 文件系统根目录下的各个文件名和想要打开的文件名，则需要 8 个字节完全匹配。

    但是值得注意的是，FAT32 文件系统根目录下的文件名并不是以 `\0` 结尾的字符串，你需要参考 spec 或者 `disk.img` 中的内容来了解 FAT32 文件名的存储方式。

**读取文件**。我们在 `fat32_read()` 函数中实现读取文件的功能。我们需要根据 `file->fat32_file` 中的信息（即 open 时获取的信息）找到文件内容所在的簇，然后读取出文件内容到 `buf` 中。

!!! tip "一些提示"
    - 通过 `cluster_to_sector()` 可以得到簇号对应的扇区号；
    - 通过 `virtio_blk_read_sector()` 可以对扇区进行读取；
    - 你可能需要再通过读取目录项来获取文件长度；
    - 读取是要从 `file->cfo` 开始的；
    - 读取的长度可能会使得内容跨越了一个或几个扇区，你需要多次读取；
    - 读取到了文件末尾就应该截止，并返回实际读取的长度；
    - 如果 read 返回了 0，则说明已经读取到了文件末尾，这样用户态程序就知道文件已经完全读取结束了。

!!! tip "关于实验中基本设定的说明"
    因为我们的文件一定在 FAT32 的根目录下，也即 `/fat32/` 下，所以我们**无需实现与目录遍历相关的逻辑**。

    此外值得注意的是，由于我们的实现是**不区分大小写**的，因此我们需要将文件名统一转换为大写或小写。

**写入文件**。我们在 `fat32_write()` 函数中实现写入文件的功能。写入操作和读取操作十分类似，我们只需要先读取出文件内容，完成修改后再调用 `virtio_blk_write_sector()` 将修改后的内容写回扇区即可。

**调整文件指针**。在写入文件时，我们可能需要先调整文件指针的位置，从某一个特定偏移开始写入，它通过 `fat32_lseek()` 函数实现。`lseek` 函数需要根据 `whence` 参数来调整 `file->cfo` 的值，你可以阅读资料了解 `lseek` 函数的具体行为。

### Task 2：实现 FAT32 文件系统

- **我们提供** `kernel/fs/virtio.c` 中的 `virtio_blk_read_sector()` 和 `virtio_blk_write_sector()` 函数来完成对磁盘扇区的读写操作，你可以直接调用它们来实现 FAT32 文件系统的功能。
- **请你实现** `kernel/fs/fat32.c` 中的 `fat32_init()`, `fat32_open_file()`, `fat32_read()`, `fat32_write()` 和 `fat32_lseek()` 函数，完成 FAT32 文件系统的初始化、文件打开、读取、写入和调整文件指针等功能。

接下来，为了让 FAT32 文件系统能够被用户态程序使用，我们需要完成上层调用者的实现：

- **请你实现** `kernel/fs/fs.c` 中的 `file_open()` 函数，它会根据传入的文件路径选择合适的文件系统（本实验中仅有 FAT32），创建对应的文件描述符对象并调用对应文件系统的 `open` 函数来打开文件。
- **请你实现** `kernel/arch/riscv/kernel/syscall.c` 中的 `sys_openat()`, `sys_close()` 和 `sys_lseek()` 函数，完成对文件的打开、关闭和调整文件指针等系统调用的实现。

!!! tip "如何判断文件系统类型"
    我们使用最简单方式来判别文件系统，文件前缀为 `/fat32/` 的即是本次 FAT32 文件系统中的文件。例如，当我们在 `nish` 中尝试读取文件，使用的命令是 `cat /fat32/$FILENAME`. `file_open()` 会根据前缀决定是否调用 `fat32_open_file` 函数。

!!! success "完成条件"
    在完成上述任务后，我们就能实现对文件的读写操作了。你可以在命令行运行如下命令尝试读取和修改 FAT32 文件系统中的文件内容：

    ``` shell
    SHELL > cat /fat32/email
    SHELL > edit /fat32/email 6 TORVALDS
    ```

    可以注意到，当我们完成修改后，文件内容是被**持久化存储**在 `disk.img` 磁盘映像文件中的，即使我们重启系统，修改后的内容依然存在。

## Part 3：从文件系统加载用户态程序

在 Lab4 中，我们实现了从内存中加载用户态 ELF 文件的功能，实现的方式是把所有用户态的 ELF 文件都打包进入 vmlinux 镜像中，与内核一同加载到内存中。

在本次实验中，我们将实现从 FAT32 文件系统中加载用户态 ELF 文件的功能。在实验提供的 `disk.img` 文件系统镜像中，包含了来自之前实验中的用户态 ELF 文件，它们分别为：

```text
nish  -- 我们提供的简单 shell 程序
print -- Lab4 中的打印程序
cow   -- Lab5 中的写时复制测试程序
fib   -- Lab5 中的斐波那契数列计算程序
mfork -- Lab5 中的多进程测试程序
```

### 进程生命周期

我们首先需要实现更加完整的进程生命周期管理，除了内核态守护进程 `kthreadd` 和第一个用户态程序 `nish` 之外，**所有进程都通过 `fork` + `execve` 的方式创建**。

其中，`fork` 系统调用的语义已经在 Lab5 中实现，接下来我们对 `execve` 系统调用的语义进行说明：

- 用户态程序通过 `execve` 系统调用来加载一个新的 ELF 文件，并将当前进程替换为该 ELF 文件对应的进程。为了简化实现，用户态的系统调用接口为 `int exec(char *path)`，没有 `argv` 和 `envp` 等其他参数。
- `execve` 系统调用会重新初始化原进程的 `vma`, `files` 等结构体，重置页表和栈空间，并将新的 ELF 文件加载到内存中，最后以恰当的方式跳转到用户态执行新程序。

为了实现父子进程和它们之间的同步关系，我们需要实现 `wait` 机制，父进程可以通过 `wait` 系统调用等待子进程的结束，并获取子进程的退出状态码。为了实现这一机制，我们需要在 `struct task_struct` 中添加新的字段：

```c
struct task_struct {
 struct task_struct *parent; /**< 父进程 */
 struct completion child_exit_comp; /**< 子进程退出的 completion */
 int exit_code; /**< 退出码 */
};
```

当我们通过 `fork` 创建子进程时，需要将子进程的 `parent` 字段指向父进程；当子进程通过 `do_exit()` 退出时，需要通过 `complete()` 唤醒等待的父进程，并将退出码存储在 `exit_code` 字段中。

当父进程调用 `wait()` 时，如果没有已经退出的子进程，则需要通过 `wait_for_completion()` 阻塞等待子进程的退出；如果有已经退出的子进程，则直接获取其退出码并返回。

!!! note "关于进程销毁的时机"
    在当前设计下，所有新进程都是通过 `fork` 的方式创建的，而 **`wait` 机制要求父进程等待子进程退出后才能销毁子进程**。因此，你需要修改 `schedule()` 中调度器的实现，把销毁进程的时机延后到父进程 `wait` 之后 (这一部分逻辑我们已经提供)。

这样一来，我们在 nish 中可以通过如下方式启动并运行新的用户态程序：

```c title="kernel/user/src/main.c"
int pid = fork();
if (pid < 0) {
    printf("Fork failed\n");
} else if (pid == 0) {
    //  子进程
    exec(exec_path);
} else {
    //  父进程
    wait(NULL);
}
```

### 从文件系统加载 ELF

为了从 FAT32 文件系统中加载 ELF 文件，我们需要修改如下两个部分的代码：

- `user_mode_thread()` 和 `load_elf_binary()`，它们的输入参数不再是内存中的 ELF 文件地址，而是一个文件路径字符串，我们需要通过文件系统打开该文件，并将其内容读入内存中；
- `do_page_fault()`，当发生缺页异常时，我们需要从文件系统中读取对应的文件内容到内存中，而不是直接从内存中的 ELF 文件拷贝。

其中，一种参考的实现接口为：

```c
struct task_struct *user_mode_thread(const char *path);
int load_elf_binary(const char *path, struct task_struct *task);
```

### Task 3：实现程序加载

请你根据上述描述，完成进程生命周期管理和从文件系统加载用户态程序的功能。

最后，请你修改 `kernel/arch/riscv/kernel/main.c`，**以 `user_mode_thread("/fat32/nish")` 的方式启动第一个用户态程序 `nish`。**

!!! tip "这一部分的自由度较大，你可以自行思考和设计具体的实现方式。"

!!! success "完成条件"
    在完成上述任务后，你的系统应当能够正确地通过 `fork` + `execve` 的方式创建并运行用户态程序，并且能够从 FAT32 文件系统中加载 ELF 文件。

    你可以在 `nish` 中运行如下命令来测试：

    ``` shell
    SHELL > run /fat32/print
    SHELL > run /fat32/cow
    SHELL > run /fat32/fib
    SHELL > run /fat32/mfork
    ```

    其运行结果应当与之前的实验一致。
