do_brk
========================================

do_brk()是一个简化版的do_mmap()，因为在这里只需要考虑匿名映射，与文件无关。
do_brk()有两个参数，addr是要扩展到的目标区域的开始地址，len是目标区域的长度。

其具体实现可参考do_mmap_pgoff,其实现如下所示:

https://github.com/leeminghao/doc-linux/blob/master/4.x.y/mm/mmap_c/do_mmap_pgoff.md

path: mm/mmap.c
```
/*
 *  this is really a simplified "do_mmap".  it only handles
 *  anonymous maps.  eventually we may be able to do some
 *  brk-specific accounting here.
 */
static unsigned long do_brk(unsigned long addr, unsigned long len)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma, *prev;
    unsigned long flags;
    struct rb_node **rb_link, *rb_parent;
    pgoff_t pgoff = addr >> PAGE_SHIFT;
    int error;

    len = PAGE_ALIGN(len);
    if (!len)
        return addr;

    flags = VM_DATA_DEFAULT_FLAGS | VM_ACCOUNT | mm->def_flags;

    error = get_unmapped_area(NULL, addr, len, 0, MAP_FIXED);
    if (error & ~PAGE_MASK)
        return error;

    error = mlock_future_check(mm, mm->def_flags, len);
    if (error)
        return error;

    /*
     * mm->mmap_sem is required to protect against another thread
     * changing the mappings in case we sleep.
     */
    verify_mm_writelocked(mm);

    /*
     * Clear old maps.  this also does some error checking for us
     */
 munmap_back:
    if (find_vma_links(mm, addr, addr + len, &prev, &rb_link, &rb_parent)) {
        if (do_munmap(mm, addr, len))
            return -ENOMEM;
        goto munmap_back;
    }

    /* Check against address space limits *after* clearing old maps... */
    if (!may_expand_vm(mm, len >> PAGE_SHIFT))
        return -ENOMEM;

    if (mm->map_count > sysctl_max_map_count)
        return -ENOMEM;

    if (security_vm_enough_memory_mm(mm, len >> PAGE_SHIFT))
        return -ENOMEM;

    /* Can we just expand an old private anonymous mapping? */
    vma = vma_merge(mm, prev, addr, addr + len, flags,
                    NULL, NULL, pgoff, NULL);
    if (vma)
        goto out;

    /*
     * create a vma struct for an anonymous mapping
     */
    vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
    if (!vma) {
        vm_unacct_memory(len >> PAGE_SHIFT);
        return -ENOMEM;
    }

    INIT_LIST_HEAD(&vma->anon_vma_chain);
    vma->vm_mm = mm;
    vma->vm_start = addr;
    vma->vm_end = addr + len;
    vma->vm_pgoff = pgoff;
    vma->vm_flags = flags;
    vma->vm_page_prot = vm_get_page_prot(flags);
    vma_link(mm, vma, prev, rb_link, rb_parent);
out:
    perf_event_mmap(vma);
    mm->total_vm += len >> PAGE_SHIFT;
    if (flags & VM_LOCKED)
        mm->locked_vm += (len >> PAGE_SHIFT);
    vma->vm_flags |= VM_SOFTDIRTY;
    return addr;
}
```