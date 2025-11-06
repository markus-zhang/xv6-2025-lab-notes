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
