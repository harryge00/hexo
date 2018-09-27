---
title: while...sleep vs tail -f /dev/null
date: 2018-09-28 00:06:51
tags:
	- Linux
	- Bash
	- Dockerfile
---
## 问题
发现用户喜欢用while ... sleep...来写Dockerfile的 ENTRYPOINT, 忍不住思考，是`tail -f /dev/null`的开销小,还是`while true
do
	sleep 3
done` 更小？


## 答案
`tail -f /dev/null` is better，因为tail -f 会一直等待 I/O 完成，进程一直处于Waiting状态。
而 while 每隔3秒会重新进行一次进程上下文切换，sleep 是单独起1个进程，而sleep结束后，会通过中断通知cpu，通过strace可以跟踪两者的系统调用

```
#strace tail -f /dev/null
execve("/usr/bin/tail", ["tail", "-f", "/dev/null"], [/* 83 vars */]) = 0
brk(NULL)                               = 0xa80000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=125541, ...}) = 0
mmap(NULL, 125541, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6f19756000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\t\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1868984, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6f19755000
mmap(NULL, 3971488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f6f19186000
mprotect(0x7f6f19346000, 2097152, PROT_NONE) = 0
mmap(0x7f6f19546000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c0000) = 0x7f6f19546000
mmap(0x7f6f1954c000, 14752, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f6f1954c000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6f19754000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6f19753000
arch_prctl(ARCH_SET_FS, 0x7f6f19754700) = 0
mprotect(0x7f6f19546000, 16384, PROT_READ) = 0
mprotect(0x60e000, 4096, PROT_READ)     = 0
mprotect(0x7f6f19775000, 4096, PROT_READ) = 0
munmap(0x7f6f19756000, 125541)          = 0
brk(NULL)                               = 0xa80000
brk(0xaa1000)                           = 0xaa1000
open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=4763056, ...}) = 0
mmap(NULL, 4763056, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6f18cfb000
close(3)                                = 0
open("/dev/null", O_RDONLY)             = 3
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
fstatfs(3, {f_type="TMPFS_MAGIC", f_bsize=4096, f_blocks=999603, f_bfree=999603, f_bavail=999603, f_files=999603, f_ffree=999080, f_fsid={0, 0}, f_namelen=255, f_frsize=4096, f_flags=4130}) = 0
lstat("/dev/null", {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
inotify_init()                          = 4
inotify_add_watch(4, "/dev/null", IN_MODIFY) = 1
stat("/dev/null", {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4, "\1\0\0\0\2\0\0\0\0\0\0\0\0\0\0\0", 26) = 16
fstat(3, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
read(3, "", 8192)                       = 0
read(4 
```
在 read(4 卡住。。。 因为I/O 无法完成读取/dev/null

```
# strace -p PID_OF_WHILE_SLEEP_SHELL
execve("./sleep.sh", ["./sleep.sh"], [/* 83 vars */]) = 0
brk(NULL)                               = 0x26ad000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=125541, ...}) = 0
mmap(NULL, 125541, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f4f4b4a1000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libtinfo.so.5", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0p\310\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=167240, ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4f4b4a0000
mmap(NULL, 2264256, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f4f4b072000
mprotect(0x7f4f4b097000, 2093056, PROT_NONE) = 0
mmap(0x7f4f4b296000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0x7f4f4b296000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\240\r\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=14608, ...}) = 0
mmap(NULL, 2109680, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f4f4ae6e000
mprotect(0x7f4f4ae71000, 2093056, PROT_NONE) = 0
mmap(0x7f4f4b070000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f4f4b070000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\t\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1868984, ...}) = 0
mmap(NULL, 3971488, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f4f4aaa4000
mprotect(0x7f4f4ac64000, 2097152, PROT_NONE) = 0
mmap(0x7f4f4ae64000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c0000) = 0x7f4f4ae64000
mmap(0x7f4f4ae6a000, 14752, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f4f4ae6a000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4f4b49f000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4f4b49e000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4f4b49d000
arch_prctl(ARCH_SET_FS, 0x7f4f4b49e700) = 0
mprotect(0x7f4f4ae64000, 16384, PROT_READ) = 0
mprotect(0x7f4f4b070000, 4096, PROT_READ) = 0
mprotect(0x7f4f4b296000, 16384, PROT_READ) = 0
mprotect(0x6f3000, 4096, PROT_READ)     = 0
mprotect(0x7f4f4b4c0000, 4096, PROT_READ) = 0
munmap(0x7f4f4b4a1000, 125541)          = 0
open("/dev/tty", O_RDWR|O_NONBLOCK)     = 3
close(3)                                = 0
brk(NULL)                               = 0x26ad000
brk(0x26ae000)                          = 0x26ae000
open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=4763056, ...}) = 0
mmap(NULL, 4763056, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f4f4a619000
close(3)                                = 0
brk(0x26af000)                          = 0x26af000
brk(0x26b0000)                          = 0x26b0000
brk(0x26b1000)                          = 0x26b1000
getuid()                                = 1000
getgid()                                = 1000
geteuid()                               = 1000
getegid()                               = 1000
rt_sigprocmask(SIG_BLOCK, NULL, [], 8)  = 0
brk(0x26b2000)                          = 0x26b2000
sysinfo({uptime=208297, loads=[19616, 40992, 43488], totalram=8243048448, freeram=258646016, sharedram=885608448, bufferram=544690176, totalswap=8467247104, freeswap=7762948096, procs=1236, totalhigh=0, freehigh=0, mem_unit=1}) = 0
brk(0x26b3000)                          = 0x26b3000
rt_sigaction(SIGCHLD, {SIG_DFL, [], SA_RESTORER|SA_RESTART, 0x7f4f4aad94b0}, {SIG_DFL, [], 0}, 8) = 0
rt_sigaction(SIGCHLD, {SIG_DFL, [], SA_RESTORER|SA_RESTART, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER|SA_RESTART, 0x7f4f4aad94b0}, 8) = 0
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], 0}, 8) = 0
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigaction(SIGQUIT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], 0}, 8) = 0
rt_sigaction(SIGQUIT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_BLOCK, NULL, [], 8)  = 0
rt_sigaction(SIGQUIT, {SIG_IGN, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
uname({sysname="Linux", nodename="haoyuan-OptiPlex-7050", ...}) = 0
brk(0x26b4000)                          = 0x26b4000
brk(0x26b5000)                          = 0x26b5000
brk(0x26b6000)                          = 0x26b6000
brk(0x26b7000)                          = 0x26b7000
brk(0x26b8000)                          = 0x26b8000
brk(0x26b9000)                          = 0x26b9000
brk(0x26ba000)                          = 0x26ba000
stat("/home/haoyuan/tmp", {st_mode=S_IFDIR|0775, st_size=4096, ...}) = 0
stat(".", {st_mode=S_IFDIR|0775, st_size=4096, ...}) = 0
getpid()                                = 14702
open("/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=26258, ...}) = 0
mmap(NULL, 26258, PROT_READ, MAP_SHARED, 3, 0) = 0x7f4f4b4b9000
close(3)                                = 0
brk(0x26bb000)                          = 0x26bb000
getppid()                               = 14700
brk(0x26bc000)                          = 0x26bc000
brk(0x26bd000)                          = 0x26bd000
getpgrp()                               = 14700
rt_sigaction(SIGCHLD, {0x447ad0, [], SA_RESTORER|SA_RESTART, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER|SA_RESTART, 0x7f4f4aad94b0}, 8) = 0
getrlimit(RLIMIT_NPROC, {rlim_cur=31086, rlim_max=31086}) = 0
brk(0x26be000)                          = 0x26be000
brk(0x26bf000)                          = 0x26bf000
rt_sigprocmask(SIG_BLOCK, NULL, [], 8)  = 0
open("./sleep.sh", O_RDONLY)            = 3
ioctl(3, TCGETS, 0x7fff135858d0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
read(3, "#!/bin/bash\nwhile true\ndo\n\tsleep"..., 80) = 40
lseek(3, 0, SEEK_SET)                   = 0
getrlimit(RLIMIT_NOFILE, {rlim_cur=1024, rlim_max=1024*1024}) = 0
fcntl(255, F_GETFD)                     = -1 EBADF (Bad file descriptor)
dup2(3, 255)                            = 255
close(3)                                = 0
fcntl(255, F_SETFD, FD_CLOEXEC)         = 0
fcntl(255, F_GETFL)                     = 0x8000 (flags O_RDONLY|O_LARGEFILE)
fstat(255, {st_mode=S_IFREG|0775, st_size=40, ...}) = 0
lseek(255, 0, SEEK_CUR)                 = 0
brk(0x26c0000)                          = 0x26c0000
read(255, "#!/bin/bash\nwhile true\ndo\n\tsleep"..., 40) = 40
stat(".", {st_mode=S_IFDIR|0775, st_size=4096, ...}) = 0
stat("/home/haoyuan/software/google-cloud-sdk/bin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/home/haoyuan/bin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/home/haoyuan/.local/bin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/usr/local/sbin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/usr/local/bin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/usr/sbin/sleep", 0x7fff13585450) = -1 ENOENT (No such file or directory)
stat("/usr/bin/sleep", 0x7fff13585450)  = -1 ENOENT (No such file or directory)
stat("/sbin/sleep", 0x7fff13585450)     = -1 ENOENT (No such file or directory)
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
geteuid()                               = 1000
getegid()                               = 1000
getuid()                                = 1000
getgid()                                = 1000
access("/bin/sleep", X_OK)              = 0
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
geteuid()                               = 1000
getegid()                               = 1000
getuid()                                = 1000
getgid()                                = 1000
access("/bin/sleep", R_OK)              = 0
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
geteuid()                               = 1000
getegid()                               = 1000
getuid()                                = 1000
getgid()                                = 1000
access("/bin/sleep", X_OK)              = 0
stat("/bin/sleep", {st_mode=S_IFREG|0755, st_size=31408, ...}) = 0
geteuid()                               = 1000
getegid()                               = 1000
getuid()                                = 1000
getgid()                                = 1000
access("/bin/sleep", R_OK)              = 0
brk(0x26c1000)                          = 0x26c1000
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14703
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGINT, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14703
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14703, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
wait4(-1, 0x7fff13584e90, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn({mask=[]})                 = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14706
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGINT, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14706
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14706, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
wait4(-1, 0x7fff13584e90, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn({mask=[]})                 = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14707
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGINT, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14707
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14707, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
wait4(-1, 0x7fff13584e90, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn({mask=[]})                 = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14708
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGINT, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14708
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14708, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
wait4(-1, 0x7fff13584e90, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn({mask=[]})                 = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14711
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigaction(SIGINT, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14711
rt_sigaction(SIGINT, {SIG_DFL, [], SA_RESTORER, 0x7f4f4aad94b0}, {0x4449b0, [], SA_RESTORER, 0x7f4f4aad94b0}, 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14711, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
wait4(-1, 0x7fff13584e90, WNOHANG, NULL) = -1 ECHILD (No child processes)
rt_sigreturn({mask=[]})                 = 0
rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_BLOCK, [INT CHLD], [], 8) = 0
clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f4f4b49e9d0) = 14712
```
