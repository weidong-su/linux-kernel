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
