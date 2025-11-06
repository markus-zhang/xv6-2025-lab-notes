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
