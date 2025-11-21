### Trial 1

My first try of rewriting `kalloc.c`. My thought was, OK I'll embed a ref count in `struct run` and then do the count. 

```C
struct run {
  // COW: every page of physical memory has a reference count (how many PTEs map to it)
  int refcount;
  struct run *next;
};

void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  // COW: reset refcount
  // NOTE: Cannot check refcount here because of freerange
  // The memory freed by freerange() is not initiated by kalloc()
  r->refcount = 0;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

There are two issues, the first one is the one shown in NOTE: since `freerange()` does not call `kalloc()`, which put refcount to 0, and `freerange()` also calls `kfree()` at the end, there is no `refcount` at all.

The second issue is way more serious. To check refcount, I need to write the function like this:

```C
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  r = (struct run*)pa;
  // COW: check refcount
  // NOTE: Assuming we don't care about the previous freerange() issue
  // The memory freed by freerange() is not initiated by kalloc()
  r->refcount = 0;

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

However, since the kernel is handing memory to the user aligned in PAGESIZE, `refcount` as part of the 4KB memory can be overwritten easily. I recall that in the memory management functions, we usually increment `va` by 4KB, and it's going to be extremely difficult, if not impossible, to make the first 8 bytes of a page readonly.

So that's why later when I checked the hint, it tells the students to use a list.

### Trial 2

OK it has been a while and I made modifications to `mappages()` to allow CoW and `kalloc.c` to allow reference counts. I also made `vmcowfault()` to handle user trap scause=15 (write page fault). I didn't change `copyout()` because I thought it is irrelevant to `simpletest()`.

These changes completely broke the kernel. After some debugging, I figured out that `copyout()` was returning -1, which caused a panic in `kexec()` when it tried to `copyout()` the CLI arguments (6 bytes for `init()`):

```C
pte = walk(pagetable, va0, 0);
    // forbid copyout over read-only user text pages.
    if((*pte & PTE_W) == 0)
      return -1;
```

What I observed is that pte at `0x87f4d018` contains the value `0x21fd2b13` -> this shows that the PA is not writable (PTE_W cleared). This is pretty weird because the CoW code is not supposed to touch it. I also confirmed that `kfork()` was never called before the panic.

I couldn't figure it out after some investigation, so I held a discussion with ChatGPT. Eventually, I realized that `mappages()` was also called in initialization, e.g. `kvmmap()` in `kvminit()` calls `mappages()`. Technically, `mappages()` distinguish kernel pages from user pages:

```C
for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("mappages: remap");
    *pte = PA2PTE(pa) | perm | PTE_V;

    // Major bug: CoW should never touch kernel pages
    if (*pte & PTE_U)
    {
      // Backup PTE_W as PTE_WB
      int pte_wbit = perm & PTE_W;
      if (pte_wbit > 0)
        *pte |= PTE_WB;

      // Clear PTE_W
      *pte &= (~PTE_W);

      // mark PTE_COW
      *pte |= PTE_COW;
      // printf("*pte: %lx\n", *pte);
    }

    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
```

And I thought this is good enough -- because CoW should never touch kernel pages, and `kfork()` is only used to fork user programs anyway.

However, I missed the point that the very first user process, `init`, is NOT created from `kfork()` -- after all, to fork from whom? Instead, it was created by the kernel directly. And of course the kernel calls `mappages()` multiple times to layout+map the memory. For example, in `kvminit()`, `kvmmake()` made a few calls to `kvmmap()` and one call to `proc_mapstacks()` (**since the panic occurred when the program is trying to write into where CLI arguments are stored, I suspect it is the stack that has permission bit issue**), each of which calls `mappages()`. Note that in those calls the program is not touching kernal pages, but user pages, because it is to setup for the first user process `init`. `mappages()` sees user pages and faithfully invoke CoW mechanisms, which probably caused the ^ issue.

What still bugs me is that I could not debug to the bottom of the issue. A lot of stuffs ^ are just guesses. I'm going to see how can I improve my debugging skills. Debugging memory issue is hard because most of the time I only have access to a memory address, or a PTE, but how do I track which function and when the PTE<->PA were mapped and when permission bits were modified? I don't want to `s` for each step.

OK I looked at the code more closely, and the ^ is probably wrong. `kvmmap()` only touches kernel pages. However, `kexec()` uses `uvmalloc()` to load the text and data sections. Using `readelf -l user/_init`, I can see the program headers. `kexec()` then uses `uvmalloc()` to allocate a guard page and a stack page.

```C
if((sz1 = uvmalloc(pagetable, sz, sz + (USERSTACK+1)*PGSIZE, PTE_W)) == 0)
    goto bad;
```

This `uvmalloc()` grows VA from `0x2000` to `0x4000`. During `uvmalloc()`, it calls `mappages()` for these two pages. **Here is where it went wrong:** (God I'm really glad that I traced to the root of the issue, can never sleep tight without knowing **WHY** something is wrong. Kernel programming is a serious business that requires us to know not only WHY but also WHY NOT)

```
va = 0x2000 -> pa: 0x87f4b000 -> pte: 0x87f4d010 -> *pte: 0x21fd2c17 (U|W|R|V bits set)
```
Note that this is user space address, so `mappages()` happily invokes the CoW logic:

```C
    // Major bug: CoW should never touch kernel pages
    if (*pte & PTE_U)
    {
      // Backup PTE_W as PTE_WB
      int pte_wbit = perm & PTE_W;
      if (pte_wbit > 0)
        *pte |= PTE_WB;

      // Clear PTE_W
      *pte &= (~PTE_W);

      // mark PTE_COW
      *pte |= PTE_COW;
      // printf("*pte: %lx\n", *pte);
    }
```

Another thing I found out:

`uvmcopy()` never touches kernel pages. This is correct. However, **guard pages** in user land has their U-bit cleared, to prevent accidental access. 


### Trial 3

The shell is not showing up. It panic in this line in `main()`. God it took me a few hours to pinpoint the exact reason. Even ChatGPT was not able to help much. I'm proud of myself to have the patience to go over every instruction/line to pinpoint the exact line that caused the issue, and then step into the kernel to find out all the bugs.

```C
      if(fork1() == 0)
        // This line panics
        runcmd(parsecmd(cmd));
      wait(0);
```

Going into `parsecmd()`, eventually I found out that in the function `parseexec()`, `**ps` was supposed to be the first char of the string `"cowtest\n"`, but somehow the string was nuked in the middle.

```C
// The line that nukes the string
  ret = execcmd();
```

`*ps`, the pointer pointing to the first char of `"cowtest\n"`, is supposed to be at the address `0x2020`. I eventually found out that `execcmd()` actually nuked the whole `0x2000` page. I stepped into `execcmd()` and found that the `malloc()` is the culprit. But I have no idea how that happened. Going into `malloc()`, I step through the code line by line, and found out the line that did the nuke.

```C
void*
malloc(uint nbytes)
{
  Header *p, *prevp;
  uint nunits;

  nunits = (nbytes + sizeof(Header) - 1)/sizeof(Header) + 1;
  if((prevp = freep) == 0){
    // The line that nukes the whole 0x2000 user page
    base.s.ptr = freep = prevp = &base;
    base.s.size = 0;
  }
```

I was then very confused, because apparently this is not a memory operation, like kfree() or something. It took me a while to learn from ChatGPT that this could be the first write instruction of the `0x2000` page. I printed out the memory addresses of these 4 items, and found out they are of `0x2088` and `0x2010`, so this is a feasible theory. Something must be wrong in `uvmcowfault()` then.

I found three bugs in the code of `uvmcowfault()`. It took me a couple of hours to step through `vmcowfault()` to figure out all of them -- till I noticed that somehow some `*pte` do not have the V bit set:

- In `vmcowfault()`:
  - Forgot to clear PTE_COW for the old pte (this is probably OK for now)
  - Forgot to decrement old pa refcount (this is bad)
  - Forgot to set the V-bit for new `*pte` (this is really bad)

I can't imagine how come I missed so many stuffs. It's like I missed half of the work! After this the shell managed to run, and I could run `simpletest()` until it got another panic -- somehow the reference count of user address 0x7000 (pa = 0x87f1b000) dropped from 2 to 0 in one shot. I have no idea how that happened, because the ONLY place to drop the refcount is in `vmcowfault()` and each time it drops the program printf a message. But I only saw one message!

```
uvmcopy va: 7000, 87f1b000, 2
vmcowfault va: 7000, 87f1b000, 0
```

After a bit of debugging, I found out that `kfree()` actually ran between the 2 lines. So this is a legit case, and I need to remove the panic code in `vmcowfault()` -- so basically, this happened:

- Step 1: 0x87f1b000 allocated by `kalloc()`
- Step 2: 0x87f1b000 got forkmapped. That's why it has a count of 2 in `uvmcopy()`
- Step 3: 0x87f1b000 was freed by `kfree()`, but because it was pointed by two PTEs, `kfree()` simply reduces the refcount.
- Step 4: 0x87f1b000 got written into again (I wonder what triggered this write, hmmm), so `vmcowfault()` panic because of the code below. I guess I have to remove it.

```C
        if (kmem.refcount[(old - KERNBASE) / PGSIZE] <= 0)
        {
          panic("vmcowfault: refcount <= 0 after pa split");
        }
```

Now finally the program passed the two `simpletest()` and the first two `threetest()`. Hmmm, not sure why it failed at the third one.

TODO: There is a vmcowfault panic: cannot allocate page from kalloc()

#### Trial 4

I couldn't figure out why the first threetest() succeeded but the third one failed, so I began another debugging session. Natually, this leads the question of whether all memory allocated has been freed properly. The number of allocation and free are just too many for me, so I put up a variable `freecount` in `kmem` and a simple system call `sys_getfreecount()` to return the variable. It's hilariously simple.

```C
extern struct kernelmem kmem;

uint64
sys_getfreecount(void)
{
  return kmem.freecount;
}
```

Then I run this sys call before and after each test function in `cowtest.c`. And found out that evenyone of them is leaking memory. More deadly, each `threetest()` leaks 4,097 pages and that's why the third one doesn't manage to allocate a page.

I can't figure it out why it is leaking, so I asked ChatGPT. Once again, without relying on it to write code for me, but relying on it to give me ideas, I found two issues:

- `uvmcopy()` is also COW RO pages, which is not needed. RO pages are not supposed to be written into, so I already had one `panic()` in `vmcowfault()` for this situation.

- `vmcowfault()` can take a shortcut if `refcount` is already one. This means that these pages are either not copied, or already split in a previous `vmcowfault()` call. The shortcut basically says, if `refcount` is 1, OK we don't have to do all those kalloc() thing, we just turn on the W-bit just in case somehow it is not there, and return the old mem.

```C
      // If refcount is already 1 (which means this pa already went through vmcowfault),
      // Then just mark it writable to make sure, and no need to split
      uint64 oldmem = PTE2PA(*pte);
      if (kmem.refcount[(oldmem - KERNBASE) / PGSIZE] == 1)
      {
          *pte |= PTE_W;
          return oldmem;
      }
```

Eventually I found out issue 2 is the culprit. The number of leaked pages are significantly smaller so all three `threetest()` passed.

I admit. The CoW logic still doesn't sound very solid. There are still some leaks, smaller ones indeed in each function call. I'm not happy about this and want to get rid of every leak. The next step is to write a bunch of debugging code to pinpoint which memory addresses are not freed (e.g. have >1 `refcount` at the final `exit()`), and step into the program to trace these memory addresses.

ChatGPT recommends a method that I really like, so I'll just log it here. It's like a kalloc() and kfree() database -- for each kalloc() or kfree(), add a line to print the index of the refcount array, and the pa. Then run `cowtest` till end. Log the whole session in a log file using `script -c "make qemu" console.log`, and then check which PA got more allocation than free.

```C
// in kfree(), towards the end
printf("Kfree idx=%ld pa=%p\n", ((uint64)pa - KERNBASE) / PGSIZE, pa);

// in kalloc(), towards the end
printf("Kalloc idx=%ld pa=%p\n", ((uint64)r - KERNBASE) / PGSIZE, (void*)r);
```

```sh
grep 'Kalloc' console.log | awk '{print $2}' | sed 's/idx=//' | sort | uniq -c > cowalloc.log
grep 'Kfree' console.log | awk '{print $2}' | sed 's/idx=//' | sort | uniq -c > cowfree.log
```

The shell commands create two files, each with contents that look like this -> First number is freq and second is index:

```
4 10000
3 10001
3 10002
3 10003
3 10004
3 10005
1 10006
3 10007
3 10008
4 10009
3 10010
```

So I quickly identified one address (a lot more): `pa=0x0000000082716000`. I'm going to see where the allocation comes from.

#### Trial 5

Well actually there is something wrong with the logging or with the shell script. None of the addresses has more allocation than free when I `printf()` each allocation and free for that PA.

I decided to write more tools for memory debugging. The next project is going to include a few functions and sys calls that can print out memory and such from the user land.