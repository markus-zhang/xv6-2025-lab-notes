The description of the lab can be found at:
https://pdos.csail.mit.edu/6.828/2025/labs/pgtbl.html

### General notes

1. PTEs of Super Pages:

   The description says, "The operating system creates a superpage by setting the PTE_V and PTE_R bits in the level-1 PTE, and setting the physical page number to point to the start of a two-megabyte region of physical memory.". It is important to understand that it is the level-1 PTE that points to the start of the physical memory, NOT the level-0 one. There is no level-0 PTE for a Super Page. It took me quite a while to understand this point, as it is not explicitly expressed in the description, and unfortunately RISC-V manual is even more obscure.

   However, the code provides a hint that points me to the right direction. I here quote the full source code of `walk()` in `vm.c`. Before reading the source code, we need to understand the concept of a *leaf* in the PTE table (or more precisely, the PDE tables). In short, leaf PTEs are the PTEs that directly point to physical memory. For regular 4KB pages, level-0 PTEs are leaf PTEs, while for 2MB super pages, level-1 PTEs are leaf PTEs. The `#ifdef` - `#endif` part says, if it is a leaf PTE, then return `pte`. This enable the `walk()` function to work for both regular 4KB pages and 2MB super pages. This part of the code "hijacks" the pte abd returns it if it's a leaf PTE for a super page, so the function doesn't walk further to fetch/create the level-0 PTE.
```C
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
if(va >= MAXVA)
  panic("walk");

for(int level = 2; level > 0; level--) {
  pte_t *pte = &pagetable[PX(level, va)];
  if(*pte & PTE_V) {
    pagetable = (pagetable_t)PTE2PA(*pte);
#ifdef LAB_PGTBL
    if(PTE_LEAF(*pte)) {
      return pte;
    }
#endif
  } else {
    if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
      return 0;
    memset(pagetable, 0, PGSIZE);
    *pte = PA2PTE(pagetable) | PTE_V;
  }
}
return &pagetable[PX(0, va)];
}
```

2. Test program

   The test program is `pgtbltest.c`. It is very important to read through it to understand what it is testing and how it expects the super page implementation to behave. I'll provide a full list of commented code of the two functions `superpg_fork()` and `supercheck()`.

```C
void
superpg_fork()
{
  int pid;
  
  printf("superpg_fork starting\n");
  testname = "superpg_fork";

  // SZ = 8 * SUPERPGSIZE =  16MB
  // hint: from where do we grow?  
  char *end = sbrk(SZ);
  if (end == 0 || end == SBRK_ERROR)
    err("sbrk failed");

  // check 1: check if parent has super pages
  supercheck(end);
  if((pid = fork()) < 0) {
    err("fork");
  } else if(pid == 0) {
   // check 2: check if child's address space has super pages
   // hint: fork() calls kfork() which calls uvmcopy() to copy the parent process' page table and physical memory
   supercheck(end);
   exit(0);
  } else {
    int status;
    wait(&status);
    if (status != 0) {
      exit(0);
    }
  }

  // free super pages
  // hint: the program allocated SZ and then deallocated SZ
  sbrk(-SZ);
  if((pid = fork()) < 0) {
    err("fork");
  } else if(pid == 0) {
    // reference freed memory; this should result in page fault and
    // the kernel should kill the child.
    * (end + 1) = '9'; 
  } else {
    int status;
    wait(&status);
    if (status == 0) {
      err("child was able to reference free memory\n");
      exit(1);
    }
  }  
  printf("superpg_fork: OK\n");  
}
```

   The second function `supercheck()` is more interesting as we can probe the mind of the of examiner.

```C
void
supercheck(char *end)
{
  pte_t last_pte = 0;
  uint64 a = (uint64) end;
  // SUPERGROUNDUP(a) returns the start of the next (aligned to 2MB)
  uint64 s = SUPERPGROUNDUP(a);

  // For each 4KB boundary address (i.e. each number that can be divided by 0x1000, e.g. 0xfff000), fetch its leaf PTE.
  // pgpte() calls walk() which returns the leaf PTE
  for (; a < s; a += PGSIZE) {
    pte_t pte = (pte_t) pgpte((void *) a);
    if (pte == 0) {
      err("no pte");
    }
  }

  printf("supercheck: check 1 done\n");
  // Within a super page, all va should have the same Level-1 PTE
  /*
     Recall that a VA under RISC-V S39 is:
     H <----- EXT | L2 | L1 | L0 | Offset ------>L
                    9    9    9    12-bit
     512 = 2 ^ 9 and 4,096 = 2 ^ 12
     This means that all VA within a super page shares the same L2 and L1 bits.
     In turn, they share the same Level-2 and Level-1 PTEs.
     We only look at the Level-1 ones because that is what walk() returns.
  */
  for (uint64 p = s;  p < s + 512 * PGSIZE; p += PGSIZE) {
    pte_t pte = (pte_t) pgpte((void *) p);
    if(pte == 0)
      err("no pte");
    if ((uint64) last_pte != 0 && pte != last_pte) {
        err("pte different");
    }
    // uvmalloc() calls mappages() to mark the V|R|W bits
    if((pte & PTE_V) == 0 || (pte & PTE_R) == 0 || (pte & PTE_W) == 0){
      err("pte wrong");
    }
    last_pte = pte;
  }

  printf("supercheck: check 2 done\n");
  // Setting and checking values at each 4KB boundary
  for(int i = 0; i < 512 * PGSIZE; i += PGSIZE){
    *(int*)(s+i) = i;
  }

  for(int i = 0; i < 512 * PGSIZE; i += PGSIZE){
    if(*(int*)(s+i) != i)
      err("wrong value");
  }

  printf("supercheck: check 3 done\n");
}
```

   The third function, juding by my experience, would be the hardest to run successfully. It checks the details of the implementation, and even a small mistake may fail this check.

```C
void
superpg_free()
{
  int pid;
  
  printf("superpg_free starting\n");
  testname = "superpg_free";

  char *end = sbrk2(SZ);
  if (end == 0 || end == SBRK_ERROR)
    err("sbrk failed");

  // free pages beyond a super page
  // This check expects the kernel programmer to fully understand alignment and allocate pages of proper size accordingly. I'll talk about it later in the next section.
  char *a = sbrk(0);
  uint64 s = SUPERPGROUNDDOWN((uint64) a);
  sbrk(-((uint64) a-s));
  a = sbrk(0);

  // Once the free pages beyond a super page have been freed, the function then exams whether the underlying memory contains a 2MB super page
  // If it is incorrectly implemented as multiple 4KB regular pages, pte1 is different from pte2.
  pte_t pte1 = (pte_t) pgpte((void *) a-PGSIZE);
  pte_t pte2 = (pte_t) pgpte((void *) a-2*PGSIZE);
  if (pte1 != pte2) {
    err("not a super page");
  }

  printf("superpg_free: test 1 done!\n");
  
  // write to the last 8192-byte section of a super page
  * (a - PGSIZE + 1) = '8';
  * (a - 2*PGSIZE + 1) = '9';

  // free last 4096 bytes of a super page
  // This is the toughest part of all checks. The kernel programmer is expected to "demote" a super page to multiple regular pages, as the description says.
  sbrk(-PGSIZE);
  a = sbrk(0);

  // "Demoting" a super page doesn't mean we should free all of its allocated physcial memory
  if (*(a - PGSIZE + 1) != '9') {
    err("lost content after freeing part of super page");
  }

  printf("superpg_free: test 2 done!\n");

  if((pid = fork()) < 0) {
    err("fork");
  } else if(pid == 0) {
     // the memory at address a shouldn't be in the child's address
     // space, since the parent freed it. The following reference
     // should result in page fault and the kernel should kill the
     // child.
    if (* (a + 1) == '9') {
      exit(0);
    }
  } else {
    int status;
    wait(&status);
    if (status == 0) {
      err("child was able to reference free memory\n");
      exit(1);
    }
  }

  printf("superpg_free: test 3 done!\n");

  // There are programmers, and there are system programmers. A programmer only needs to vaguely knows what "free" means. A system programmer needs to understand the concept with clarity. A kernel programmer needs to implement it.
  pte1 = (pte_t) pgpte((void *) a);
  if(pte1 != 0) {
    err("pte for freed memory is valid");
  }

  printf("superpg_free: test 4 done!\n");

  s = SUPERPGROUNDDOWN((uint64) a);
  for (; (uint64) a > s; a -= PGSIZE) {
    a = sbrk(-PGSIZE);
    pte1 = (pte_t) pgpte(sbrk(0));
    if(pte1 != 0) {
      err("page hasn't been freed");
    }
  }

  printf("superpg_free: test 6 done!\n");
  
  printf("superpg_free: OK\n");  
}
```

3. Kernel debugging notes

   I'm far from a programmer who has excellent debugging skills, but I have learned much using `gdb`.

   First, anyone who attempts the lab should read this webpage and setup the environment first: https://pdos.csail.mit.edu/6.828/2025/tools.html

   Please note that certain Linux distros, such as Ubuntu LTS, does not have the required version. I had to manually install QEMU 7.X to run xv6 in a virtual machine.

   I found it is useful to use `~/.gdbinit` to preload some configurations for each session. Here is my very short file:

```bash
set auto-load local-gdbinit on
set auto-load safe-path /
add-symbol-file ./user/_pgtbltest
layout src    
```

   The second line enables xv6 debugging, otherwise gdb does not attach to the kernel, at least it was my experience. The third line loads the symbols of `pgtbltest.c` so that I can break in this file. The last line switches the layout to `src`. You might find other useful options, but this is good enough for me at the moment.

   I'm still trying to figure out some issues I encountered in debugging. I suspect they have something to do with compiler optimization (which causes compiler to skip over source code).
   - Sometimes gdb skips code. gdb never breaks on assignment, which I get and am fine with it, but sometimes it seems to skip important lines.
   - Sometimes gdbbreaks at a place (e.g. `b pgtbltest.c:xxx`) without the program being run at all. It could break at the place even before shell shows up, and I so far haven't figured out the why. I had to input `c` multiple times, to reach the real breakpoint.

   Some other tips when debugging user land. 
   - xv6 context switches between kernel and user for each system call. Simply stepping through each step does not work as gdb loses track during the context switch. I guess `layout asm` might work to some extend but I never validated that.
   - What I did is to recognize each system call function -- most of the time it is the `sys_???` function, and set a breakpoint at that function. Once `ecall` is executed, gdb should break at exactly the place you want it to.
   - To debug memory allocation and deallocation, we need to understand how xv6 allocate/deallocate memory, how it organizes phsyical memory, and how it maps virtual addresses to physcial addresses. I'll discuss my findings in the next chapter.


### Memory allocation overview

To understand memory allocation of a user process, it is essential to step through the whole process from its birth to its death. Except for the initial user process, every other process, including the shell, starts from `kfork()` call from its parent process. Just a note that I may skip some function calls and focus on the more important ones.

xv6 uses paging, which means each allocation and deallocation are aligned at 4KB boundary. This is why in some of the original comments and code you will see a check against alignment, and `panic` if the addresses (virtual or physical ones) are not. Some functions do not expect the caller to watch alignment and we will talke about them one by one later.

```C
int
kfork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  // uvmcopy() copies BOTH the page table as well as the physical memory the entries point to
  // This means if the parent process allocates 16MB of memory, its child process would need approximately the same amount of memory during `kfork()`
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  /*
    From my understanding, the reasons kfork() nees to do a full copy of the memory of the parent process are:
    - The scheduler doesn't care whether it is a parent process or a child process, it picks any process that is RUNNABLE
    - if the child process misses important components, such as stack, then it crashes immediately
    - so it's kinda easiest just to do a full copy

    Questions though, like, can we simply inject just enough code/data to make a very simple program? Why do we need to do a full copy anyway?
  */

  // increment reference counts on open file descriptors.
  // I don't understand this part yet, gotta keep going
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  // Just to copy name, so if I `p p->name` and `p np->name`, they are the same. Confused me the hell when I just started.
  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  release(&np->lock);

  acquire(&wait_lock);
  np->parent = p;
  release(&wait_lock);

  acquire(&np->lock);
  // So the scheduler could pick np
  np->state = RUNNABLE;
  release(&np->lock);

  return pid;
}
```

I'm going to ignore as many secondary function call as possible, as we will discuss some of them when we start working on the implementation of Super Page. Once `kfork()` is done, the scheduler kicks into work, and eventually Shell runs `kexec()` for the child process. `exec()` is where the actual text and data sections and other stuffs get allocated and mapped.

```C
//
// the implementation of the exec() system call
//
int
kexec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  // Open the executable file.
  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // Read the ELF header.
  /*
    For ELF file, I use `readelf` to inspect the executable. For example, `readelf -h` shows the file header, while `readelf -l` shows the program headers, etc.
    Assuming cwd is xv6-labs-2025 and you already ran `make qemu` once without issues.
    $ cd user
    $ readelf -h _pgtbltest
    The ^ shell command gives the ELF header information. A more interesting switch is:
    $ readelf -l _pgtbltest

    Elf file type is EXEC (Executable file)
    Entry point 0x56c
    There are 4 program headers, starting at offset 64

    Program Headers:
    Type           Offset             VirtAddr           PhysAddr
                   FileSiz            MemSiz              Flags  Align
    RISCV_ATTRIBUT 0x000000000000ab81 0x0000000000000000 0x0000000000000000
                   0x0000000000000033 0x0000000000000000  R      0x1
    LOAD           0x0000000000001000 0x0000000000000000 0x0000000000000000
                   0x0000000000001211 0x0000000000001211  R E    0x1000
    LOAD           0x0000000000003000 0x0000000000002000 0x0000000000002000
                   0x0000000000000010 0x0000000000000030  RW     0x1000
    GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                   0x0000000000000000 0x0000000000000000  RW     0x10

    Section to Segment mapping:
    Segment Sections...
    00     .riscv.attributes 
    01     .text .rodata 
    02     .data .bss 
    03

    You can see the text and data sections in the Program Headers list as two "LOAD" type. How can I decide which one is text and which one is data? We can tell by reading the "Flags". text section is executable but not writable, while data is writable but not executable, so the first LOAD section is the text section (code), while the second LOAD section is the data section. The text section starts at Virtual Address 0x0 and has a size of 0x1211, while the data section starts at 0x2000 and has a size of 0x30.

    Now we must take a detour and read the user address space map to get an idea about how the process address grows:
    MAXVA-------> --------------------
                  |   trampoline     |
                  --------------------
                  |   trapframe      |
                  --------------------
                  |                  |
                  |                  |
                  |     unused       |
                  |                  |
                  --------------------
                  |                  |
                  |                  |
                  |       heap       |
                  |                  |
                  --------------------
    One 4KB Page->|       stack      |
                  --------------------
                  |    guard page    |
                  --------------------
                  |                  |
                  |                  |
                  |       data       |
                  |                  |
                  --------------------
                  |      unused      |
                  --------------------
                  |                  |
                  |                  |
                  |       text       |
                  |                  |
    0-----------> --------------------

    So in the case of _pgtbltest, the text section takes 2  pages. It only needs 0x1211 bytes but since it goes over one page it has to take two full pages. The data section then starts from 0x2000 and take just one page because it is so small.
    The rest of kexec() does the work of loading sections into memory.
                  
  */
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;

  // Is this really an ELF file?
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    // If it's not a LOAD section, we don't load (check the map ^)
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    uint64 sz1;
    /*
      I'll go over uvmalloc() in the next chapter because we need to modify it. Adding super pages makes allocation and deallocation much more complicated.
      In summary, uvmalloc does two things:
      1. call kalloc() to allocate physical memory
      2. call mappages() to map virtual addresses to physical addresses
      Essentially, it performs the service so that the next time the OS access a VA, it can trace it to the PA and do actual stuffs.
    */
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz, flags2perm(ph.flags))) == 0)
      goto bad;
    sz = sz1;
    // Load program segments, nothing fancy here
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate some pages at the next page boundary.
  // Make the first inaccessible as a stack guard.
  // Use the rest as the user stack.

  // PGROUNDUP() is frequently used to bring a VA to the starting address of next page, because of alignment.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  // It allocates 2 pages, one for user stack and one for the guard page
  if((sz1 = uvmalloc(pagetable, sz, sz + (USERSTACK+1)*PGSIZE, PTE_W)) == 0)
    goto bad;
  sz = sz1;
  // This is how it setup the guard page. Guard page is just a glorious name for a cleared page. OS screams when it is told to touch a cleared page, so it serves as a "guard" page against bad VA.
  uvmclear(pagetable, sz-(USERSTACK+1)*PGSIZE);
  // sp is where stack pointer should be, and stackbase is the base of stack. Stack grows from high address to low address, so sp > stackbase.
  sp = sz;
  stackbase = sp - USERSTACK*PGSIZE;

  // Copy argument strings into new stack, remember their
  // addresses in ustack[].
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push a copy of ustack[], the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // a0 and a1 contain arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.

  // The ^ is just calling conventions
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = ulib.c:start()
  p->trapframe->sp = sp; // initial stack pointer
  // OS code is pretty resilience so we need to free our own shit
  proc_freepagetable(oldpagetable, oldsz);

  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```
