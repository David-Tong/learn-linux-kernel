# cd、open 和 read 如何找到文件：从系统调用到 inode

本文以 Linux 7.0、ext4 普通文件为例，解释路径如何变成内存中的 inode，以及 `read()` 如何使用已经打开的文件。挂载如何把 ext4 接入 VFS，请先看[挂载和 VFS 的连接](mount-inode-file-lookup.md)。具体函数随内核版本、架构和文件系统变化；以下调用链省略了权限检查、错误处理和若干中间步骤。

## 先区分三个操作

| 用户操作 | 内核要做什么 | 查找的起点 |
| --- | --- | --- |
| shell 的 `cd /mnt/data/docs` | 通常调用 `chdir()`，解析目录并更新当前工作目录 | 绝对路径从进程的根目录开始 |
| `open("a.txt", O_RDONLY)` | 解析路径，打开文件，返回 fd | 相对路径通常从当前工作目录开始 |
| `read(fd, buf, n)` | 根据 fd 找到已打开的文件并读取 | fd 表；不重新解析路径 |

`cd` 是 shell 内建命令，因为它必须改变 shell 自己的当前目录，才能影响之后启动的程序。`chdir()` 和 `fchdir()` 是 POSIX 接口，也是 Linux 系统调用；`read()` 读取文件描述符，不接收文件名。shell 可能先处理 `~`、逻辑路径和 `CDPATH`，所以命令行文本不一定原样传给内核。`cd -` 等行为还有 shell 自己的处理。

## cd 和 chdir：把当前目录改成一个路径对象

### shell 如何从 `cd` 走到 `chdir()`

以 Bash 中的 `cd docs` 为例，回车后，shell 先解析命令行，把 `cd` 识别为内建命令，再把 `docs` 作为参数交给内建命令的实现。Bash 根据参数和选项处理目标路径，例如展开 `~`、考虑 `CDPATH`，并根据 `-L` 或 `-P` 处理逻辑路径与物理路径。处理后的路径不一定与用户输入的文本相同。

随后，Bash 在 **shell 自己的进程**中调用 C 库的 `chdir(path)` 接口。C 库通过系统调用入口将路径传给 Linux 内核；内核完成路径查找和权限检查后，更新该进程的当前工作目录。成功后 shell 更新 `PWD`、`OLDPWD` 等状态；失败时报告错误，当前工作目录不变。具体实现和调用方式因 shell、C 库及运行环境而异。

当前目录属于进程：如果单独启动一个子进程来执行 `chdir()`，改变的只是子进程的目录，不能反过来改变父 shell。因此交互式 shell 通常把 `cd` 做成内建命令。之后从该 shell 启动的程序会继承当前目录；例如 `cd /etc` 后执行 `cat passwd`，`passwd` 这个相对路径通常从 `/etc` 开始查找。在 `(cd /etc)` 这样的子 shell 中执行则只影响子 shell。

### Bash 怎样通过系统调用与内核交互

Bash 的 `builtins/cd.def` 定义了 `cd_builtin()`。它处理选项和参数，并经由 `cd_to_string()`、`change_to_directory()` 等函数选择和处理目录；具体分支可能尝试 `CDPATH` 中的多个候选目录。最终的目录切换会调用 `chdir()`，下图省略了路径规范化和错误处理等分支：

```text
Bash: cd_builtin → cd_to_string → change_to_directory
  → chdir(目标路径)                 Bash 调用 C 库接口
  → glibc 系统调用封装             按架构约定准备参数和调用号
  → syscall 指令                   从用户态进入内核（x86-64）
  → Linux chdir 系统调用处理       查找目录并设置当前工作目录
  → 返回用户态                     Bash 根据结果更新状态或报告错误
```

以 Linux x86-64 和 glibc 为例，Bash 把以 `\0` 结尾的路径字符串放在自己的用户空间内存中，并把指针传给 `chdir()`。系统调用封装把 `chdir` 的系统调用号放入 `rax`，路径指针放入 `rdi`，执行 `syscall` 指令。内核通过入口代码接收调用号和参数，再安全地从用户空间获取路径；Bash 不能通过这个指针直接修改内核的目录对象。其他架构的寄存器和入口指令可能不同。

内核处理完成后返回结果：成功为 `0`；失败时内核返回负错误码，glibc 将其转换为 `-1` 并设置 `errno`。Bash 检查结果，成功后更新自己的工作目录相关状态，失败时可用 `errno` 报告原因，例如目录不存在或没有搜索权限。内核维护的当前目录才是后续相对路径查找的依据；`PWD` 是 shell 维护的路径表示。

可以直接观察用户态到内核的边界：

```bash
strace -e trace=chdir bash -c 'cd /etc'
# 典型输出：chdir("/etc") = 0
strace -e trace=chdir bash -c 'cd /this-directory-does-not-exist'
# 典型输出包含：chdir(...) = -1 ENOENT (No such file or directory)
```

`strace` 显示系统调用及其结果；它不会显示 Bash 内部的函数调用，也不会显示内核内部的路径查找函数。

### 内核如何处理 `chdir()`

以 `chdir("/mnt/data/docs")` 为例，Linux 7.0 在 `fs/open.c` 中的系统调用主体负责取得路径、查找目标、检查权限并更新当前目录。路径查找本身由 `fs/namei.c` 中的 VFS 代码完成：

```text
SYSCALL_DEFINE1(chdir, ...)                fs/open.c
  → 准备用户传入的路径名
  → filename_lookup(...)                  查找目标路径
      → VFS 逐个解析 mnt、data、docs       fs/namei.c
      → 返回目标的 struct path
  → path_permission(...)                  检查进入目标目录的权限
  → set_fs_pwd(current->fs, &path)         更新当前工作目录
  → path_put(&path)                        释放查找时持有的引用
```

箭头表示主要过程，并非全部是直接调用。查找或权限检查失败时，内核返回错误，不更新当前目录。`fchdir(fd)` 则从已打开的目录 fd 获得路径，不需要再次按名字解析这个目录。

Linux 7.0 的 `fs/open.c` 中，`chdir` 系统调用主体主要完成以下工作：

- **准备路径名**：`CLASS(filename, name)(filename)` 将用户空间传入的路径名交给内核的文件名管理机制，并在作用域结束时清理资源。用户指针无效或路径不合法时，后续查找会返回错误。
- **查找目录**：`filename_lookup(AT_FDCWD, name, lookup_flags, &path, NULL)` 把路径解析为 `struct path`。相对路径以当前工作目录为起点，绝对路径以进程根目录为起点。`LOOKUP_FOLLOW` 要求跟随最后一个符号链接，`LOOKUP_DIRECTORY` 要求最终目标是目录。逐级查找的细节见后文的 VFS 路径查找。
- **检查权限**：`path_permission(&path, MAY_EXEC | MAY_CHDIR)` 检查是否允许进入目标目录；路径解析过程中也会检查沿途目录的搜索权限。
- **更新当前目录**：检查成功后，`set_fs_pwd(current->fs, &path)` 更新当前任务文件系统上下文中的 `pwd` 路径引用。这个操作建立对新路径的引用并处理旧路径的引用，不是单纯修改 `PWD` 字符串。
- **释放与返回**：`path_put(&path)` 释放本次查找持有的临时引用。遇到适合重试的 `ESTALE` 时，代码加入 `LOOKUP_REVAL` 重新查找；最终返回 `0` 或负错误码。

可以把系统调用主体概括为下面的伪代码，细节以 [Linux 7.0 源码](https://github.com/torvalds/linux/blob/v7.0/fs/open.c)为准：

```c
name = 从用户路径准备内核文件名(filename);
flags = LOOKUP_FOLLOW | LOOKUP_DIRECTORY;

retry:
error = filename_lookup(AT_FDCWD, name, flags, &path, NULL);
if (error == 0) {
    error = path_permission(&path, MAY_EXEC | MAY_CHDIR);
    if (error == 0)
        set_fs_pwd(current->fs, &path);
    path_put(&path);

    if (retry_estale(error, flags)) {
        flags |= LOOKUP_REVAL;
        goto retry;
    }
}
return error;
```

当前目录是进程文件系统上下文中的路径引用。绝对路径从进程的根目录开始，该根目录可能受容器或 `chroot` 环境影响。

## VFS 如何逐级把名字解析成 inode

### `filename_lookup()` 里的 VFS 路径查找

`filename_lookup()` 定义在 `fs/namei.c`，负责组织路径查找及必要的重试；逐级遍历路径的主要工作交给 `path_lookupat()`。在 `chdir()` 中的调用为：

```c
filename_lookup(AT_FDCWD, name, lookup_flags, &path, NULL);
```

- `AT_FDCWD`：相对路径从当前工作目录开始；绝对路径从进程根目录开始。
- `name`：内核准备好的路径名；如果它是错误指针，函数直接返回对应错误码。
- `lookup_flags`：查找要求；这里包括跟随最后一个符号链接和要求结果是目录。
- `&path`：成功时写入结果，包括挂载对象和 dentry。
- `NULL`：调用者没有另外指定查找根目录。

函数首先通过 `set_nameidata()` 准备 `struct nameidata` 查找上下文，再调用 `path_lookupat()`。它先带 `LOOKUP_RCU` 尝试 RCU-walk；如果返回 `-ECHILD`，去掉该标志重试，使用引用计数方式查找。如果返回 `-ESTALE`，则带 `LOOKUP_REVAL` 重试以重新验证。成功时调用 `audit_inode()` 进行审计处理，最后通过 `restore_nameidata()` 恢复上下文。成功返回 `0`，失败返回负错误码；`chdir()` 成功找到路径后还要检查权限并设置当前目录。

#### 从 `filename_lookup()` 到 `link_path_walk()`

在 [Linux 7.0 的 `fs/namei.c`](https://github.com/torvalds/linux/blob/v7.0/fs/namei.c) 中，两者之间经过 `path_lookupat()`：

```text
filename_lookup()
  → set_nameidata()                 准备查找上下文 nd
  → path_lookupat(&nd, flags, path)
      → path_init(nd, flags)        确定查找起点，返回待解析的路径
      → link_path_walk(s, nd)       处理路径分量，检查目录搜索权限
          → may_lookup()            检查当前目录能否搜索
          → walk_component()        查找并进入中间分量
      → lookup_last(nd)             处理最后一个分量
      → complete_walk(nd)           完成查找并验证结果
      → 将 nd->path 交给输出参数 path
```

`path_init()` 对绝对路径通常选择进程根目录；对于 `AT_FDCWD` 的相对路径选择当前工作目录。`link_path_walk()` 处理前面的分量并在 `nd` 中记录最后一个分量；`path_lookupat()` 随后通过 `lookup_last()` 处理它。若最后一个分量是需要跟随的符号链接，`lookup_last()` 可返回链接目标，循环会再次调用 `link_path_walk()`。图中展示主要路径，省略了错误、挂载点和其他分支；查找失败或需要重试时不会直接返回成功结果。

VFS 路径查找是 `filename_lookup()` 完成的工作，不是一个名为“VFS 路径查找”的单独函数。它根据路径名从起点出发，逐个找到目录项，并在需要时让具体文件系统查找名字：

```text
/mnt/data/docs
  /          从进程根目录出发
  → mnt      在根目录中查找 mnt
  → data     在 mnt 中查找 data；若是挂载点，跨入被挂载的文件系统
  → docs     在该文件系统的根目录中查找 docs
  → path     得到目标的挂载对象和 dentry
```

对于每个分量，VFS 检查当前目录能否被搜索，尝试利用 dentry 缓存找到名字；缓存不能满足查找时，走合适的慢路径，通过父目录 inode 的 `i_op->lookup` 让所属文件系统查找。以 ext4 为例，可能进入 `ext4_lookup()`，从目录内容找到 inode 号并取得对应的内存 inode。缓存命中时不一定调用文件系统的 `lookup`，查找也不一定产生磁盘 I/O。

#### 搜索权限与 dentry 缓存

目录的 `x` 位表示搜索（穿越）权限。例如解析 `/mnt/data/docs`，需要依次搜索 `/`、`/mnt` 和 `/mnt/data`；对于 `chdir()`，找到 `docs` 后还要检查能否进入目标目录。内核结合进程凭据和目录的所有者、所属组、权限位判断访问，也可能涉及 ACL、进程的 capability 和安全模块；不是只要任意一组权限中有 `x` 就允许搜索。

目录的 `r` 位主要控制能否列出目录项，`x` 位控制能否按名字搜索或穿越。即使没有 `r`，知道名字且有搜索权限时，也可能访问该名字对应的文件；访问文件本身仍须通过相应权限检查。缓存命中也不能绕过目录搜索权限检查。

Linux 7.0 中路径遍历对中间目录的搜索权限检查，主要位于 [`fs/namei.c`](https://github.com/torvalds/linux/blob/v7.0/fs/namei.c)：

```text
link_path_walk()                         逐个处理路径分量
  → may_lookup()                         检查当前目录能否搜索
    → lookup_inode_permission_may_exec() 检查目录的 MAY_EXEC 权限
      → 快速检查，或进入 inode_permission() 等完整权限检查
```

`lookup_inode_permission_may_exec()` 是针对目录遍历的优化检查：它将 `MAY_EXEC` 加入请求。如果文件系统允许使用该快速路径、目录的三组 `x` 权限均已设置且没有 ACL，就可以跳过较复杂的普通权限判断，但仍调用安全模块检查。其他情况会进入 `inode_permission()` 等完整检查：文件系统可提供自己的 `inode->i_op->permission`，否则通常使用 `generic_permission()`，结合权限位、ACL 和进程凭据判断，必要时也考虑 capability。

对于 `chdir()` 的最终目标，路径查找成功后，[`fs/open.c`](https://github.com/torvalds/linux/blob/v7.0/fs/open.c) 还会调用 `path_permission(&path, MAY_EXEC | MAY_CHDIR)`，检查能否进入该目录；通过后才更新当前工作目录。

例如在 `/mnt/data` 中查找 `docs`，VFS 根据父目录和名字查找 dentry 缓存：

- 有效的正 dentry 关联找到的 inode，可以继续使用该结果。
- 有效的负 dentry 缓存了该名字不存在的结果。
- 缓存不能满足查找时，VFS 在适当的慢路径调用父目录 inode 的 `i_op->lookup`；ext4 可能通过 `ext4_lookup()` 查找目录内容并取得目标 inode。

如果遇到挂载点，VFS 会跨入挂载的文件系统；遇到符号链接则按查找标志解析。不存在的分量、不可搜索的目录或非目录目标等都会导致错误。成功后，`struct path` 中的挂载对象与 dentry 共同确定目标位置，dentry 关联目标目录的 inode。`chdir()` 再检查并保存这个路径，而不是保存输入字符串。下面以普通文件为例继续解释相同的查找机制。

假设 `/dev/vdb1` 上的 ext4 挂载在 `/mnt/data`，解析 `/mnt/data/docs/a.txt`：

```text
进程根目录 → mnt → data [跨过挂载点] → ext4 根目录 → docs → a.txt
                                                    ↑ 父目录     ↑ 目标
```

### ext4 如何找到 inode

以前面的 `chdir("/mnt/data/docs")` 为例，假设 `/mnt/data` 是 ext4 的挂载点，查找 `docs` 时，VFS 已经跨入 ext4 并把该文件系统的根目录作为父目录。如果 dentry 缓存不能满足查找，可能沿下面的调用链进入 ext4：

```text
chdir() → filename_lookup() → path_lookupat()        fs/open.c、fs/namei.c
  → link_path_walk()                                 处理前面的路径分量
  → lookup_last()                                    处理最后的 docs
    → walk_component()
      → lookup_fast()                                尝试 dentry 缓存
      → lookup_slow()                                快速查找未得到结果时
        → __lookup_slow()                            协调并发查找
          → 父目录 inode->i_op->lookup(...)
            → ext4_lookup()                          fs/ext4/namei.c
              → 在 ext4 目录数据中查找 docs 的 inode 号
              → ext4_iget()                          获取目标内存 inode
              → 关联目标 dentry 与 inode
```

这里 `link_path_walk()` 和 `lookup_last()` 是 `path_lookupat()` 中先后执行的步骤；若查找的是中间分量，`link_path_walk()` 也可以直接经由 `walk_component()` 进入同一条慢路径。路径图展示的是一种可能的执行过程，不表示每次查找都会调用 ext4：缓存命中或慢路径中复用并发查找结果时可以不调用 `ext4_lookup()`。

为什么函数指针会指向 ext4？当 ext4 加载目录 inode 时，它会把该 inode 的 `i_op` 设置为 `ext4_dir_inode_operations`；这个操作表的 `.lookup` 成员指向 `ext4_lookup`。因此 VFS 不需要直接写死 ext4 函数名，只需通过**父目录** inode 的操作表调用。参见 [ext4 inode 初始化](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)和 [ext4 目录操作表](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)。

`ext4_lookup()` 在目录数据中查找目标名字对应的 inode 号，通过 `ext4_iget()` 获取内存 inode，再处理它与目标 dentry 的关联。inode 不在缓存时，需要从磁盘 inode 表加载；查找目录本身也可能产生 I/O。不存在的名字则可能形成负 dentry。

这是简化图：负 dentry 可以缓存不存在的名字，符号链接会引入额外解析，`O_CREAT` 等操作对最后一个分量有特殊处理；缓存命中时也不必调用 ext4 的 `lookup`。目录搜索权限、挂载跨越、并发重命名及 RCU-walk / REF-walk 都会影响实际过程。硬链接使多个 dentry 指向同一个 inode；inode 号只在所属文件系统内有意义。

ext4 使用 inode 号和每块组 inode 数算出块组及组内索引，通过块组描述符定位磁盘 inode 表。磁盘 inode 是元数据记录；内存 `struct inode` 是 VFS 对象，由 ext4 初始化其操作表。加载 inode 不意味着已经读取全部文件内容。

## open：路径解析后，建立打开文件和 fd

```c
int fd = open("a.txt", O_RDONLY);
```

这是 POSIX `open()` 库接口。在 Linux 上 libc 可能以 `openat(AT_FDCWD, ...)` 实现；`strace` 看到 `openat` 不代表应用源码一定调用了 `openat()`。`openat()` 的相对路径可从目录 fd 开始，`AT_FDCWD` 则使用当前工作目录。

```text
open() / openat()                用户态接口
  → openat 系统调用入口           fs/open.c
  → do_sys_openat2() → do_file_open()
  → path_openat()                fs/namei.c；解析路径、检查权限并打开
  → struct file → fd 表项
  → 返回 fd
```

### `path_openat()` 怎样检查沿途目录权限

以 `open("/mnt/data/docs/a.txt", O_RDONLY)` 打开已有普通文件为例，Linux 7.0 的普通打开分支会经过下面的步骤：

```text
do_file_open()
  → path_openat()
    → path_init()                         确定路径查找起点
    → link_path_walk()                    逐级处理路径分量
      → may_lookup()                      检查当前目录的搜索权限
        → lookup_inode_permission_may_exec()  请求 MAY_EXEC 检查
      → walk_component()                  查找并进入中间分量
    → open_last_lookups()                 处理最后的 a.txt
    → do_open()                           检查目标并完成打开
```

`link_path_walk()` 遍历时，在当前目录查找下一个名字之前调用 `may_lookup()`。因此这个例子通常要依次有权搜索 `/`、`/mnt`、`/mnt/data` 和 `/mnt/data/docs`；没有权限时查找停止，打开通常返回 `EACCES`。这里的搜索权限对应目录的 `x` 位，但判断还会考虑进程凭据、ACL、capability 和安全模块。具体机制与前面 [搜索权限与 dentry 缓存](#搜索权限与-dentry-缓存)相同。

这条普通打开路径由 `path_openat()` 直接调用 `link_path_walk()`，不必经过 `filename_lookup()`；而 `chdir()` 是通过 `filename_lookup()`、`path_lookupat()` 到达同一个遍历函数。`link_path_walk()` 记录最后一个分量，普通打开流程随后用 `open_last_lookups()` 处理它；符号链接和 `O_CREAT` 等可能引入其他分支，`O_PATH` 等标志也有专门路径。

沿途目录的搜索检查与目标文件的打开检查是两件事：找到 `a.txt` 后，内核还会根据 `O_RDONLY` 等访问方式检查目标文件，通常需要读取权限。若使用 `O_CREAT` 创建文件，还需要满足父目录相应的写入和搜索权限。详见 [Linux 7.0 `fs/namei.c`](https://github.com/torvalds/linux/blob/v7.0/fs/namei.c)。

`openat2()` 也会进入相关打开流程，但其解析限制和参数与 `openat()` 不同。针对普通文件，打开过程把已解析的路径和 inode 与新的 `struct file` 关联，并通常从 `inode->i_fop` 取得文件操作表；文件系统打开方法可以调整它。两个独立的 `open()` 即使指向同一 inode，也通常得到各自的打开文件状态和偏移；`dup()` 则共享同一个打开文件描述及偏移。

## read 从哪里继续？

`read(fd, buf, count)` 使用 fd 找到打开时保存的 `struct file`，不重新按路径查找 inode。读取如何通过 VFS 和文件操作表进入 ext4、页缓存和块 I/O，见 [VFS 如何调用文件系统的读取实现](vfs-filesystem-read-dispatch.md)。文件描述符共享偏移的实验见 [Bash 文件描述符参考](references/bash-file-handle-analysis.md)。

## 亲自观察

在 Linux 上运行以下命令，观察 shell 及其子进程的系统调用：

```bash
strace -f -e trace=chdir,fchdir,openat,read bash -c 'cd /etc && cat passwd > /dev/null'
```

输出会包含动态加载器和其他文件的 `openat`、`read`；找到 `chdir("/etc")`、打开 `passwd` 以及对应 fd 的 `read` 即可。`cat` 可能使用其他优化路径；系统、libc 和工具版本也会影响结果。`strace` 展示用户态与内核的边界，不直接展示 `ext4_lookup()` 等内部函数，也不能证明发生了磁盘 I/O。

可以比较路径和已打开 fd 引用的 inode：

```bash
stat -c 'device=%d inode=%i %n' /etc/passwd
bash -c 'exec 3</etc/passwd; stat -Lc "device=%d inode=%i %n" /proc/$$/fd/3'
```

设备号和 inode 号应相同；这种观察展示对象身份，但不直接展示内核操作表或缓存状态。

## 源码与参考

- [Bash 的 cd 实现](https://git.savannah.gnu.org/cgit/bash.git/tree/builtins/cd.def)：内建命令和目录切换。
- [Linux x86-64 系统调用表](https://github.com/torvalds/linux/blob/v7.0/arch/x86/entry/syscalls/syscall_64.tbl)：系统调用号与入口。
- [Linux 7.0 fs/open.c](https://github.com/torvalds/linux/blob/v7.0/fs/open.c)：`chdir`、`fchdir` 和打开入口。
- [Linux 7.0 fs/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/namei.c)：路径查找和打开。
- [Linux 7.0 fs/read_write.c](https://github.com/torvalds/linux/blob/v7.0/fs/read_write.c)：`read` 系统调用和 VFS 读取。
- [Linux 7.0 ext4/namei.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/namei.c)、[ext4/inode.c](https://github.com/torvalds/linux/blob/v7.0/fs/ext4/inode.c)：ext4 查找及 inode 加载。
- [内核路径查找文档](https://docs.kernel.org/7.0/filesystems/path-lookup.html)、[VFS 文档](https://docs.kernel.org/7.0/filesystems/vfs.html)。
