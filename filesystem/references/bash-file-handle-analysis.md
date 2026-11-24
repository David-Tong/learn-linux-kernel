# Bash 文件描述符：inode、打开文件与 fdinfo

在 Bash 中执行 `exec 3< file` 后，`3` 是当前进程的文件描述符（fd）。它引用内核中的打开文件对象 `struct file`，该对象再关联文件的 inode。对普通文件而言，**读取偏移量属于打开文件对象，而不是 inode，也不是 fd 数字本身**。

本文以 Linux 7.0 和 Bash 为例；有关 inode 如何加载，可继续阅读 [路径查找与系统调用](../path-lookup-syscalls.md)。

## 1. fd、struct file 和 inode

```text
Bash 进程的文件描述符表
   fd 3 ──→ struct file（一次打开的文件描述）
               ├─ f_pos：当前文件偏移
               ├─ f_flags：访问模式和文件状态标志
               └─ f_inode ──→ inode（文件系统对象）
                                   ├─ 文件类型、权限、大小等元数据
                                   └─ 文件数据的映射信息
```

fd 是进程文件描述符表的索引。Linux 的 `struct file` 对应通常所说的 **open file description（打开文件描述）**；inode 表示文件系统中的对象，其大小、权限等信息由同一文件的打开实例共同关联。文件名通过目录项关联到 inode；不同名字也可能通过硬链接指向同一 inode。

```bash
exec 3< /etc/passwd   # 在当前 Bash 中打开，以 fd 3 引用
exec 3<&-             # 关闭 fd 3
```

`exec` 没有命令参数时，重定向作用于当前 shell。Bash 可能需要打开后再调整 fd 编号，具体系统调用序列取决于现有描述符和 Bash 实现。0、1、2 通常分别用作标准输入、输出和错误，但也可以指向文件、管道等对象。

## 2. `/proc/$$/fd` 和 `fdinfo`

`$$` 在普通 Bash 脚本或交互式 shell 中展开为该 shell 的 PID。procfs 按需展示内核状态，这些路径不是保存 fd 状态的普通磁盘文件：

下面逐行执行，观察读取前后的变化。示例使用 fd 3，请先确认它没有被当前 shell 用于其他用途：

```bash
tmp=$(mktemp)
printf 'one\ntwo\nthree\n' > "$tmp"
exec 3<"$tmp"

readlink /proc/$$/fd/3    # 查看 fd 3 关联的文件
cat /proc/$$/fdinfo/3    # 读取前：pos 为 0

IFS= read -r -u 3 line
echo "$line"              # one
cat /proc/$$/fdinfo/3    # 读取一行后：pos 为 4

IFS= read -r -u 3 line
echo "$line"              # two
cat /proc/$$/fdinfo/3    # 读取两行后：pos 为 8

IFS= read -r -u 3 line
echo "$line"              # three
cat /proc/$$/fdinfo/3    # 读取三行后：pos 为 14

exec 3<&-
rm -f -- "$tmp"
```

`pos` 按**字节**计数，并包括每行结尾的换行符：例如 `one\n` 是 4 字节。这里的内容都是 ASCII，且 Bash 对这个可定位的普通文件按行读取；其他程序可能预读数据，使内核偏移比应用已经处理的内容更靠后。

读取第二行后，普通文件的 `fdinfo` 输出可能是：

```text
pos:    8
flags:  0100000
mnt_id: 29
ino:    123456
```

| 字段 | 含义 |
| --- | --- |
| `pos` | 十进制文件偏移；普通文件中通常是下一次顺序读写开始的字节位置，对应打开文件对象的 `f_pos`。 |
| `flags` | 八进制访问模式和文件状态标志；还会反映该 fd 的 `FD_CLOEXEC`。具体值因打开方式和平台而异。 |
| `mnt_id` | 该打开文件关联的挂载 ID，可在 `/proc/$$/mountinfo` 中查找对应挂载。 |
| `ino` | inode 号；inode 号只在所属文件系统中标识对象，不能在所有文件系统之间单独比较。 |

例如 `grep '^29 ' /proc/$$/mountinfo` 可查上例的挂载信息；实际应替换为看到的 `mnt_id`。`readlink` 的输出用于展示对象，并非稳定身份：文件可能被重命名或删除，管道、socket 等也不一定显示普通路径。`fdinfo` 字段会随对象类型增加；管道和 socket 的 `pos` 不应解释为普通文件读写偏移。详情见 [proc_pid_fdinfo(5)](https://man7.org/linux/man-pages/man5/proc_pid_fdinfo.5.html)和 [proc_pid_fd(5)](https://man7.org/linux/man-pages/man5/proc_pid_fd.5.html)。

## 3. 实验：同一个 inode，独立打开与复制 fd

在 Linux 的 Bash 中逐行执行下面的命令。示例使用 fd 3、4、5，先确认当前 shell 没有将它们用于其他用途：

```bash
tmp=$(mktemp)
printf 'abcdef' > "$tmp"

exec 3<"$tmp"         # 第一次打开
exec 4<"$tmp"         # 独立打开同一个文件
exec 5<&3             # 复制 fd 3
```

先查看三个 fd 的信息。它们的 `ino` 相同，初始 `pos` 都是 0：

```bash
cat /proc/$$/fdinfo/3
cat /proc/$$/fdinfo/4
cat /proc/$$/fdinfo/5
```

从 fd 3 读取两个 ASCII 字符，再查看偏移：

```bash
IFS= read -r -n 2 -u 3 data
echo "$data"           # ab

cat /proc/$$/fdinfo/3  # pos: 2
cat /proc/$$/fdinfo/4  # pos: 0
cat /proc/$$/fdinfo/5  # pos: 2
```

fd 3 和 fd 5 共享打开文件描述，所以偏移一起变化；fd 4 是独立打开的，偏移仍为 0：

```text
fd 3 ──┐
       ├──→ struct file A：f_pos = 2 ──┐
fd 5 ──┘                                ├──→ 同一个 inode
fd 4 ─────→ struct file B：f_pos = 0 ──┘
```

实验结束后关闭 fd 并删除临时文件：

```bash
exec 3<&- 4<&- 5<&-
rm -f -- "$tmp"
```

两次独立 `open()` 创建两个打开文件描述，所以各有偏移；复制 fd 与 `dup()` 类似，两个 fd 引用同一个打开文件描述，所以共享偏移和文件状态标志。`fork()` 后父子进程继承的对应 fd 也引用相同的打开文件描述；关闭其中一个 fd 不会关闭其他引用。参见 [open(2)](https://man7.org/linux/man-pages/man2/open.2.html)和 [dup(2)](https://man7.org/linux/man-pages/man2/dup.2.html)。

相同的 `ino`、`pos` 或 `flags` **不能单独证明**两个 fd 共享同一个 `struct file`：独立打开的 fd 也可能恰好有相同值。上面的共享关系来自已知的复制操作，偏移变化只是可观察到的结果。示例读取限定为 ASCII；Bash `read -n` 按字符处理，多字节文本不能直接把字符数当字节数。

## 4. 同一个进程能把同一个文件打开两次吗？同时写入会怎样？

可以。每次独立调用 `open()` 都会得到一个新的 `struct file`；它们关联同一个 inode 和同一份文件内容，但各有自己的 `f_pos`。下面分三种情况观察。示例在 Linux 的 Bash 中逐行执行，使用 fd 3 和 4；请先确认它们没有被当前 shell 使用。

### 情况一：独立打开，写入可能覆盖

先准备一个空文件，并分别打开两次：

```bash
tmp=$(mktemp)
exec 3<>"$tmp"       # 第一次打开，偏移为 0
exec 4<>"$tmp"       # 第二次打开，偏移也为 0
```

从 fd 3 写入后，比较两个偏移：

```bash
printf 'AAAA' >&3
cat /proc/$$/fdinfo/3  # pos: 4
cat /proc/$$/fdinfo/4  # pos: 0
```

再从 fd 4 写入：

```bash
printf 'BB' >&4        # 从自己的偏移 0 开始写
cat "$tmp"             # BBAA
cat /proc/$$/fdinfo/3  # pos: 4
cat /proc/$$/fdinfo/4  # pos: 2
```

这两个写入是**顺序执行**的，结果可预测。如果不同线程或进程同时写入，范围可能重叠，最终内容取决于写入顺序和范围。独立偏移不会隔离文件内容。

关闭并清理：

```bash
exec 3>&- 4>&-
rm -f -- "$tmp"
```

### 情况二：复制 fd，共享偏移

重新创建文件，打开一次，然后复制 fd 3：

```bash
tmp=$(mktemp)
exec 3<>"$tmp"
exec 4>&3             # fd 4 与 fd 3 指向同一个 struct file

printf 'AAAA' >&3
cat /proc/$$/fdinfo/3  # pos: 4
cat /proc/$$/fdinfo/4  # pos: 4

printf 'BB' >&4       # 从共享的偏移 4 继续写
cat "$tmp"             # AAAABB
cat /proc/$$/fdinfo/3  # pos: 6
cat /proc/$$/fdinfo/4  # pos: 6

exec 3>&- 4>&-
rm -f -- "$tmp"
```

对于 Linux 7.0 普通文件，共享打开文件描述的并发 `write()` 会原子地更新共享偏移，避免仅因偏移更新竞争而写到同一范围；但写入先后顺序仍不确定。多个 `write()` 组成的业务记录也不保证作为整体连续。

### 情况三：独立打开，但每次追加到文件末尾

```bash
tmp=$(mktemp)
exec 3>>"$tmp"         # 独立打开，设置 O_APPEND
exec 4>>"$tmp"         # 再次独立打开，也设置 O_APPEND

printf 'AAAA' >&3
printf 'BB' >&4        # 写入时定位到当时的文件末尾
cat "$tmp"              # AAAABB

cat /proc/$$/fdinfo/3  # pos: 4
cat /proc/$$/fdinfo/4  # pos: 6
exec 3>&- 4>&-
rm -f -- "$tmp"
```

对支持该语义的本地文件系统，每次 `write()` 的定位到末尾和写入是一个原子步骤。并发追加的顺序仍不确定；一条记录若分成多次 `write()`，其他写入可能穿插其中，`write()` 也可能只写入部分数据。NFS 等文件系统可能有不同限制；`O_APPEND` 也不等于 `fsync()` 的持久化保证。协调复杂更新或完整记录时，应使用所有参与者共同遵守的锁或其他同步机制。详见 [open(2)](https://man7.org/linux/man-pages/man2/open.2.html)和 [write(2)](https://man7.org/linux/man-pages/man2/write.2.html)。

## 5. 几个容易混淆的细节

- 普通 `read()` 成功读取数据后通常推进共享的 `f_pos`；`lseek()` 可以修改可定位文件的偏移。`pread()` 使用显式偏移，不改变共享的文件偏移。到达文件末尾时，普通 `read()` 返回 0。
- 文件内容或大小属于底层文件对象的状态；独立打开只让偏移等打开状态独立，并不复制一份文件内容。文件被重命名或解除最后一个名字后，已有打开文件仍可引用原对象，直到最后引用释放。
- `FD_CLOEXEC` 属于**描述符**，不是共享的文件状态标志。设置它后，该 fd 在成功执行 `execve()` 时关闭；单纯 `fork()` 不会因此关闭它。`fdinfo` 的 `flags` 会把它显示为 `O_CLOEXEC` 位。
- `/proc/self` 指访问 procfs 的进程：用外部 `cat /proc/self/fdinfo/3` 时，`self` 指 `cat`，不一定是想检查的 Bash。上面的 `/proc/$$` 明确指向 Bash；在 Bash 子 shell 中 `$$` 仍可能是父 shell 的 PID，需要本进程 PID 时可使用 `BASHPID` 并注意展开时机。
