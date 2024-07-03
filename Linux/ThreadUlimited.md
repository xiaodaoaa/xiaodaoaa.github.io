在 Linux 系统中，每个进程能够创建的线程数量主要受以下几个因素限制：

### 1. 系统可用资源
每个线程都会消耗一定的系统资源，包括内存、文件描述符等。因此，系统的总资源限制了进程能创建的线程数量。

### 2. 用户和系统级限制
系统有一些配置参数和限制，可以控制单个进程可以创建的最大线程数：

#### a. `ulimit -u`
`ulimit` 命令可以显示和设置用户级限制。`ulimit -u` 选项显示或设置每个用户可以创建的最大进程数（包括线程，因为线程在 Linux 中被认为是轻量级进程）。

查看当前用户最大进程数限制：
```sh
ulimit -u
```

设置当前用户最大进程数限制（例如设置为 4096）：
```sh
ulimit -u 4096
```

#### b. `/proc/sys/kernel/threads-max`
这个文件包含系统级的最大线程数限制。可以通过读取该文件查看系统允许的最大线程数：
```sh
cat /proc/sys/kernel/threads-max
```

可以通过写入该文件设置系统级的最大线程数限制：
```sh
echo 100000 > /proc/sys/kernel/threads-max
```

#### c. `/proc/sys/vm/max_map_count`
这个文件控制了每个进程可以拥有的内存映射区域的最大数量。线程栈是通过内存映射来实现的，所以这个限制也会影响线程数量。
```sh
cat /proc/sys/vm/max_map_count
```

#### d. `/proc/sys/kernel/pid_max`
这个文件控制了系统中进程 ID 的最大值。因为线程也有 PID，所以这个限制也会影响线程数量：
```sh
cat /proc/sys/kernel/pid_max
```

#### e. `rlimit`
在编程中，可以使用 `setrlimit` 函数设置资源限制，包括进程和线程数量。

### 3. 内存限制
线程栈占用内存，因此系统可用内存也限制了可创建的线程数。默认情况下，每个线程的栈大小为 8MB，但可以通过 `ulimit -s` 查看和设置：

查看当前线程栈大小：
```sh
ulimit -s
```

设置线程栈大小（例如设置为 2MB）：
```sh
ulimit -s 2048
```

### 示例

查看当前用户最大进程数限制：
```sh
ulimit -u
```

查看系统级的最大线程数限制：
```sh
cat /proc/sys/kernel/threads-max
```

查看最大内存映射区域数：
```sh
cat /proc/sys/vm/max_map_count
```

查看进程 ID 最大值：
```sh
cat /proc/sys/kernel/pid_max
```

综上所述，单个进程能创建的最大线程数取决于系统资源、用户级和系统级限制以及内存限制。调整这些参数可以改变最大线程数的限制。