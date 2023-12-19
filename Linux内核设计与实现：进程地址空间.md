进程的地址空间，就是每个进程所使用的内存。内核除了管理自身的内存，还管理用户空间中进程的内存。
# 地址空间
进程地址空间就是进程能访问的内存地址范围。尽管进程可以寻址4GB的虚拟内存，但并不代表它有权访问所有的虚拟地址。这些可被进程访问的地址区间，成为内存区域。
如果进程访问了不在有效范围的内存区域，就会报错“段错误”。
内存区域主要包含各种内存对象，比如：
- 代码段：可执行文件代码的内存映射
- 数据段：可执行文件中已初始化的全局/静态变量的内存映射
- BSS数据段：可执行文件中未初始化的全局/静态变量的内存映射（页面信息都为0，bss全称block started by symbol）
- 栈段：包含局部变量和函数调用的上下文。栈的大小是固定的，一般是8MB。
- 堆段：动态分配的内存，从低地址开始生长
- 文件映射段：包含动态库、共享内存等，从低地址开始向上生长
堆段和文件映射段是动态分配的内存，C标准库的malloc函数和mmap函数分别可以在堆段和文件映射段动态分配内存。

# 内存描述符
内核用内存描述符来表示进程地址空间。
```
struct mm_struct {
    struct vm_area_struct * mmap;        /* [内存区域]链表 */
    struct rb_root mm_rb;               /* [内存区域]红黑树 */
    struct vm_area_struct * mmap_cache;    /* 最近一次访问的[内存区域] */
    unsigned long (*get_unmapped_area) (struct file *filp,
                unsigned long addr, unsigned long len,
                unsigned long pgoff, unsigned long flags);  /* 获取指定区间内一个还未映射的地址，出错时返回错误码 */
    void (*unmap_area) (struct mm_struct *mm, unsigned long addr);  /* 取消地址 addr 的映射 */
    unsigned long mmap_base;        /* 地址空间中可以用来映射的首地址 */
    unsigned long task_size;        /* 进程的虚拟地址空间大小 */
    unsigned long cached_hole_size;     /* 如果不空的话，就是 free_area_cache 后最大的空洞 */
    unsigned long free_area_cache;        /* 地址空间的第一个空洞 */
    pgd_t * pgd;                        /* 页全局目录 */
    atomic_t mm_users;            /* 使用地址空间的用户数 */
    atomic_t mm_count;            /* 实际使用地址空间的计数， (users count as 1) */
    int map_count;                /* [内存区域]个数 */
    struct rw_semaphore mmap_sem;   /* 内存区域信号量 */
    spinlock_t page_table_lock;        /* 页表锁 */

    struct list_head mmlist;        /* 所有地址空间形成的链表 */

    /* Special counters, in some configurations protected by the
     * page_table_lock, in other configurations by being atomic.
     */
    mm_counter_t _file_rss;
    mm_counter_t _anon_rss;

    unsigned long hiwater_rss;    /* High-watermark of RSS usage */
    unsigned long hiwater_vm;    /* High-water virtual memory usage */

    unsigned long total_vm, locked_vm, shared_vm, exec_vm;
    unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
    unsigned long start_code, end_code, start_data, end_data; /* 代码段，数据段的开始和结束地址 */
    unsigned long start_brk, brk, start_stack; /* 堆的首地址，尾地址，进程栈首地址 */
    unsigned long arg_start, arg_end, env_start, env_end; /* 命令行参数，环境变量首地址，尾地址 */

    unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

    struct linux_binfmt *binfmt;

    cpumask_t cpu_vm_mask;

    /* Architecture-specific MM context */
    mm_context_t context;

    /* Swap token stuff */
    /*
     * Last value of global fault stamp as seen by this process.
     * In other words, this value gives an indication of how long
     * it has been since this task got the token.
     * Look at mm/thrash.c
     */
    unsigned int faultstamp;
    unsigned int token_priority;
    unsigned int last_interval;

    unsigned long flags; /* Must use atomic bitops to access the bits */

    struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
    spinlock_t        ioctx_lock;
    struct hlist_head    ioctx_list;
#endif
#ifdef CONFIG_MM_OWNER
    /*
     * "owner" points to a task that is regarded as the canonical
     * user/owner of this mm. All of the following must be true in
     * order for it to be changed:
     *
     * current == mm->owner
     * current->mm != mm
     * new_owner->mm == mm
     * new_owner->alloc_lock is held
     */
    struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
    /* store ref to file /proc/<pid>/exe symlink points to */
    struct file *exe_file;
    unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
    struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
```
mmap、mm_rb描述的对象都是相同的：都是该地址空间中全部的内存区域。分别使用链表和红黑树进行存放。链表可以简单高效的遍历元素，红黑树可以高效的搜索指定元素。
## mm_struct的操作
```
allocate_mm() 
free_mm()
```
进程创建时，在fork中调用copy_mm，子进程通过allocate_mm从slab缓存中分配一个新的mm_struct。线程没有独立的地址空间，因此在copy_mm中，线程将task_struct中的mm指针指向父进程的mm
进程退出后，exit_mm会执行常规的撤销工作，更新统计量。如果mm_count计数为0，则调用free_mm将mm_struct结构体归还到slab缓存中。

查看进程占用的内存
```
pmap PID 或者
cat /proc/PID/maps
```

# 虚拟内存区域
内存区域在linux内核中被称为虚拟内存区域（virtual memory areas, VMAs），指在进程地址空间中一段连续的内存范围，其实就是对应内存区域的分段（堆段、栈段、代码段、数据段、bss段、文件映射段），用vm_area_struct描述。

## VMA
```
struct vm_area_struct {
    struct mm_struct * vm_mm;    /* 相关的 mm_struct 结构体 */
    unsigned long vm_start;        /* 内存区域首地址 */
    unsigned long vm_end;        /* 内存区域尾地址 */

    /* linked list of VM areas per task, sorted by address */
    struct vm_area_struct *vm_next, *vm_prev;  /* VMA链表 */

    pgprot_t vm_page_prot;        /* 访问控制权限 */
    unsigned long vm_flags;        /* 标志 */

    struct rb_node vm_rb;       /* 树上的VMA节点 */
    union {
        struct {
            struct list_head list;
            void *parent;    /* aligns with prio_tree_node parent */
            struct vm_area_struct *head;
        } vm_set;

        struct raw_prio_tree_node prio_tree_node;
    } shared;
    struct list_head anon_vma_node;    /* Serialized by anon_vma->lock */
    struct anon_vma *anon_vma;    /* Serialized by page_table_lock */

    /* Function pointers to deal with this struct. */
    const struct vm_operations_struct *vm_ops;

    /* Information about our backing store: */
    unsigned long vm_pgoff;        /* Offset (within vm_file) in PAGE_SIZE
                       units, *not* PAGE_CACHE_SIZE */
    struct file * vm_file;        /* File we map to (can be NULL). */
    void * vm_private_data;        /* was vm_pte (shared mem) */
    unsigned long vm_truncate_count;/* truncate_count or restart_addr */

#ifndef CONFIG_MMU
    struct vm_region *vm_region;    /* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
    struct mempolicy *vm_policy;    /* NUMA policy for the VMA */
#endif
};
```
[vm_start, vm_end]表示内存区域的有效范围。vm_mm指向与VMA相关的mm_struct结构。每个VMA结构对相关的mm_struct结构都是唯一的。所以两个独立进程将同一个文件映射到各自的进程地址空间，也会各自都有一个VMA来标识自己的内存区域。
### VMA 标志
vm_area_struct结构中的vm_flags，标识内存区域所包含页面的行为和信息。

标志 对VMA及其页面的影响
VM_READ	页面可读取
VM_WRITE	页面可写
VM_EXEC	页面可执行
VM_SHARED	页面可共享
VM_MAYREAD	VM_READ 标志可被设置
VM_MAYWRITER	VM_WRITE 标志可被设置
VM_MAYEXEC	VM_EXEC 标志可被设置
VM_MAYSHARE	VM_SHARE 标志可被设置
VM_GROWSDOWN	区域可向下增长
VM_GROWSUP	区域可向上增长
VM_SHM	区域可用作共享内存
VM_DENYWRITE	区域映射一个不可写文件
VM_EXECUTABLE	区域映射一个可执行文件
VM_LOCKED	区域中的页面被锁定
VM_IO	区域映射设备I/O空间
VM_SEQ_READ	页面可能会被连续访问
VM_RAND_READ	页面可能会被随机访问
VM_DONTCOPY	区域不能在 fork() 时被拷贝
VM_DONTEXPAND	区域不能通过 mremap() 增加
VM_RESERVED	区域不能被换出
VM_ACCOUNT	该区域时一个记账 VM 对象
VM_HUGETLB	区域使用了 hugetlb 页面
VM_NONLINEAR	该区域是非线性映射的

### VMA 操作

```
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs. 
 */
struct vm_operations_struct {
    void (*open)(struct vm_area_struct * area);  /* 指定内存区域加入到一个地址空间时，该函数被调用 */
    void (*close)(struct vm_area_struct * area); /* 指定内存区域从一个地址空间删除时，该函数被调用 */
    int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf); /* 当没有出现在物理页面中的内存被访问时，该函数被调用 */

    /* 当一个之前只读的页面变为可写时，该函数被调用，
     * 如果此函数出错，将导致一个 SIGBUS 信号 */
    int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);

    /* 当 get_user_pages() 调用失败时, 该函数被 access_process_vm() 函数调用 */
    int (*access)(struct vm_area_struct *vma, unsigned long addr,
              void *buf, int len, int write);
#ifdef CONFIG_NUMA
    /*
     * set_policy() op must add a reference to any non-NULL @new mempolicy
     * to hold the policy upon return.  Caller should pass NULL @new to
     * remove a policy and fall back to surrounding context--i.e. do not
     * install a MPOL_DEFAULT policy, nor the task or system default
     * mempolicy.
     */
    int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);

    /*
     * get_policy() op must add reference [mpol_get()] to any policy at
     * (vma,addr) marked as MPOL_SHARED.  The shared policy infrastructure
     * in mm/mempolicy.c will do this automatically.
     * get_policy() must NOT add a ref if the policy at (vma,addr) is not
     * marked as MPOL_SHARED. vma policies are protected by the mmap_sem.
     * If no [shared/vma] mempolicy exists at the addr, get_policy() op
     * must return NULL--i.e., do not "fallback" to task or system default
     * policy.
     */
    struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
                    unsigned long addr);
    int (*migrate)(struct vm_area_struct *vma, const nodemask_t *from,
        const nodemask_t *to, unsigned long flags);
#endif
};
```
