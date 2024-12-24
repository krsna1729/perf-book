\phantomsection
# Appendix B. Enable Huge Pages {.unnumbered}

\markboth{Appendix B}{Appendix B}

## Windows {.unnumbered}

To utilize huge pages on Windows, you need to enable `SeLockMemoryPrivilege` [security policy](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/lock-pages-in-memory). This can be done programmatically via the Windows API, or alternatively via the security policy GUI.

1. Hit start &rarr; search "secpol.msc", and launch it.
2. On the left select "Local Policies" &rarr; "User Rights Assignment", then double-click on "Lock pages in memory".

![Windows security: Lock pages in memory](../../img/appendix-C/WinLockPages.png){width=100%}

3. Add your user and reboot the machine.

4. Check that huge pages are used at runtime with [RAMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/rammap) tool.

Use huge pages in the code with:

```cpp
void* p = VirtualAlloc(NULL, size, MEM_RESERVE | 
                                   MEM_COMMIT | 
                                   MEM_LARGE_PAGES,
                       PAGE_READWRITE);
...
VirtualFree(ptr, 0, MEM_RELEASE);
```

## Linux {.unnumbered}

On Linux OS, there are two ways of using huge pages in an application: Explicit and Transparent Huge Pages.

### Explicit Huge Pages {.unnumbered .unlisted}

Explicit huge pages can be reserved at system boot time or before an application starts. To make a permanent change to force the Linux kernel to allocate 128 huge pages at the boot time, run the following command:

```bash
$ echo "vm.nr_hugepages = 128" >> /etc/sysctl.conf
```

To explicitly allocate 128 huge pages after the system has booted, you can use the following command:

```bash
$ echo 128 > /proc/sys/vm/nr_hugepages
```

You should be able to observe the effect in `/proc/meminfo`. Note that it is a system-wide view and not per-process:

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i"
AnonHugePages:      2048 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:     128    <== 128 huge pages allocated
HugePages_Free:      128
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:          262144 kB <== 256MB of space occupied
```

You can utilize explicit huge pages in the code by calling `mmap` with the `MAP_HUGETLB` flag ([full example](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c)[^25]):

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
...
munmap(ptr, size);
```

Other alternatives include:

* `mmap` using a file from a mounted `hugetlbfs` filesystem ([example code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c)[^26]).
* `shmget` using the `SHM_HUGETLB` flag ([example code](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c)[^27]).

### Transparent Huge Pages {.unnumbered .unlisted}

To allow applications to use Transparent Huge Pages (THP) on Linux you should ensure that `/sys/kernel/mm/transparent_hugepage/enabled` is `always` or `madvise`. The former enables system-wide usage of THPs, while the latter gives control to the user code on which memory regions should use THPs, thus avoiding the risk of consuming more memory resources. Below is an example of using the `madvise` approach:

```cpp
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
madvise(ptr, size, MADV_HUGEPAGE);
...
munmap(ptr, size);
```

You can observe the system-wide effect in `/proc/meminfo` under `AnonHugePages`:

```bash
$ watch -n1 "cat /proc/meminfo  | grep huge -i" 
AnonHugePages:     61440 kB     <== 30 transparent huge pages are in use
HugePages_Total:     128
HugePages_Free:      128        <== explicit huge pages are not used
```

Also, you can observe how your application utilizes EHPs and/or THPs by looking at the `smaps` file specific to your process:

```bash
$ watch -n1 "cat /proc/<PID_OF_PROCESS>/smaps"
```

[^25]: MAP_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/map_hugetlb.c).
[^26]: Mounted `hugetlbfs` filesystem - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-mmap.c).
[^27]: SHM_HUGETLB example - [https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/vm/hugepage-shm.c).
