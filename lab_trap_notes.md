### Trap

#### Trial 0

`sys_sigalarm()` should be pretty straightforward. It needs to receive two arguments n and fn. If both are 0, it zeros `alarmon` variable (which controls whether the alarm is on). Otherwise it sets `alarmon` to 1, `alarmticks` to 0, and `alarmtriggerticks` to `n`. It also sets `alarmfp` to `fn` for future `periodic()` call.

**Crucial**: `sys_sigalarm()` is only called once for each `sigalarm()` user space function call, so we cannot rely on it to *update* `alarmticks`.

**Crucial**: From what we see in the lab doc, the `for` loop that prints `.` keeps running while something updates `alarmticks` in the background.


**First experiment**:

Use something in **kernel space** to update `alarmticks` and somehow calls `fn` as a function:

I picked `scheduler()` because it runs processes by slices. It increments `alarmticks` and compare it with `alarmtriggerticks`. Once equal or greater than, it replaces `epc` with `alarmfp`, which is `fn`, the address to the function `periodic()`.

```C
// in scheduler(void)
        if (p->alarmon == 1)
        {
          p->alarmticks += 1;
          if (p->alarmticks >= p->alarmtriggerticks)
          {
            // WHY: Next line only works if I modify epc, not ra, why? Actually, ra occasionally works, just takes too long
            p->trapframe->epc = p->alarmfp;
            p->alarmticks = 0;
          }
        }
```

This does show some promises so I went on to implement `sys_sigreturn()`. It's a bit tricky as we apparently cannot allow it to return normally to execute `printf()` and then `exit()`. Eventually I got an idea: **I'll save the user space return address (`p->trapframe->ra`) of the syscall SYS_sigalarm, BEFORE it went into kernel space. i.e. saves it in syscall(). Then, if there is a SYS_sigreturn syscall, I replace the then user space return address with the saved return address**.

```C
// in syscall(void)
  // Alarmtest debugging
  if (num == SYS_sigalarm)
  {
    p->alarmreturnaddress = p->trapframe->ra;
  }

  if (num == SYS_sigreturn)
  {
    p->trapframe->ra = p->alarmreturnaddress;
  }

```

`sys_sigreturn()` is actually an empty function because it doesn't need to do anything. It just returns 0.

**Con**: It passes test0 somewhat, but immediately triggers a instruction page fault (`sscause == 0xc`). I don't know why. `sepc` shows 0x14f50, and I traced the program to find out that it was loaded from the stack frame during the last step of returning to user space (in `trampiline.S` where `ld ra, 40(a0)` is executed).

**Guess**: Actually, how does it work? Why does the `for` loop gets to run? I found out that `test0()` works if I commented out the `write` line.

```
usertrap(): unexpected scause 0xc pid=5
            sepc=0x14f50 stval=0x14f50
```

#### Trial 1

OK so it took me a long time to figure this out. There are two suspects from the beginning. The first suspect is that I vaguely feel that changing the `epc` is not equivalent to a function call, so maybe I should do something? But somehow this just scrubs against my mind and I didn't bother to check it out. The second suspect is to figure out `0x14f50`.

`0x14f50` doesn't exactly look like an instruction, and when I checked the user land stack using `x/32x $sp`, I can see this number popping up in the stack frame.

I actually learned a lot about gdb debugging during this period of time, so not every hour is wasted. Here are some tricks:

- `info line *$ra` to figure out which line it returns;

- Before `sret`, use `b *$ra` to break right at the first instruction after returning to user land. I need to do it after the user land `ra` is reloaded

- `define hook-stop` but somehow this seems to trigger `si` twice, not sure why

Anyway, eventually I resorted to debugging in assembly, so I got to learn how to properly debug between kernel land and user land too, as a bonus.

Then I got an idea from ChatGPT which says I shouldn't write the code in `scheduler()` because there might be some race situation. I actually don't believe it because as far as printf debugging shows, the scheduler and the alarmtest processes are the only two running. But I moved the code to `usertrap()` anyway. It didn't work, as I expected. The reason I like `scheduler()` is, it is the only thing I can think of that stops/starts a process automatically. I completely forgot about the timer interrupt. TBH I probably thought something needs to call the timer interrupt.

Eventually after so many trials, I gave up and decided to read the hints in the lab. First, it does recommend using `usertrap()`, so that's alright. Next, it hints about saving some registers. Well that was actually what I did in `sys_sigreturn()` and `usertrap()` -- basically, `usertrap()` saves `ecp` and `sys_sigreturn()` restores it, but apparently it didn't work, so I'm completely out of ideas now. I decided to research what are caller preserved registers and save all of them, which didn't work either. That `0x14f50` keeps showing up. From what I see in gdb, it shows up from the beginning of `test0()` so it's probably some leftover from its caller function.

So I asked ChatGPT a question: if I change `sepc` manually pointing to a function `blah()`, when is `blah()` executed? Somehow I never figured out clearly what the manual means ("When a trap is taken into S-mode, sepc is written with the virtual address of the instruction that was interrupted or that encountered the exception."), because there are multiple function calls within the kernel land for each system call (trampoline.S -> usertrap() -> syscall() -> ...) so I got confused what does this "insutrction that was interrupted" actually means. Well actually it just means the first user land instruction when the process returns to the user land.

In the same answer ChatGPT also told me that there is a caveat of doing this, because it is not a proper function call so there is no stack frame. I was not familiar with the assembly code, so I wonder what that means. I know what a stack frame is, but I always thought that the callee function sets it up, so why bother maually sets up a stack frame even if we are executing the callee function using `epc`? But then I took look of the assembly code of `periodic()` and astonishly found out that it didn't do much!

```asm
void
periodic()
{
   0:	1141                	addi	sp,sp,-16
   2:	e406                	sd	ra,8(sp)
   4:	e022                	sd	s0,0(sp)
   6:	0800                	addi	s0,sp,16
  s11_var = ~S11_CHECKVAL;
   8:	7df9                	lui	s11,0xffffe
   a:	7dad8d9b          	addiw	s11,s11,2010 # ffffffffffffe7da <base+0xffffffffffffd7ca>
  count = count + 1;
   e:	00001797          	auipc	a5,0x1
  12:	ff27a783          	lw	a5,-14(a5) # 1000 <count>
  16:	2785                	addiw	a5,a5,1
  18:	00001717          	auipc	a4,0x1
  1c:	fef72423          	sw	a5,-24(a4) # 1000 <count>
  printf("alarm!\n");
  20:	00001517          	auipc	a0,0x1
  24:	c5050513          	addi	a0,a0,-944 # c70 <malloc+0xe4>
  28:	2b1000ef          	jal	ra,ad8 <printf>
  sigreturn();
  2c:	728000ef          	jal	ra,754 <sigreturn>
  printf("oops, sigreturn returned!\n");
  30:	00001517          	auipc	a0,0x1
  34:	c4850513          	addi	a0,a0,-952 # c78 <malloc+0xec>
  38:	2a1000ef          	jal	ra,ad8 <printf>
  exit(1);
  3c:	4505                	li	a0,1
  3e:	66e000ef          	jal	ra,6ac <exit>

0000000000000042 <slow_handler>:
  }
```

So what we can see here is, it touches sp, s0, s11, a5 and a bunch of stuffs, but never bothered to restore them. I did restore most of them, but I didn't restore `sp`, and this turned out to be the real issue.

I think there might be some value, to write an OS COMPLETELY in assembly. No one does anything extra for you in assembly, so you gotta be really careful, and know every nitty-gritties of the internal stuffs. And TBH risc-v asm doesn't seem to be too bad. Maybe I'll "port" xv6 to assembly later as a project, using the asm file as a reference.

#### Trial 2

Now we are in test1, and looks like `a0` is not preserved. I spent an hour debugging this thing and found the reason:

In `sys_sigreturn()`, I did restore `p->trapframe->a0`, which shows 1337. But right after the return it is erased, because `a0` is used as the register that contains the return value -- wait! I'm wrong. We are talking about `p->trapframe->a0`, not `a0` itself, hmmmm...

OK I found it. When `sys_sigreturn()` returns to `syscall()`, the return value is stored in `p->trapframe->a0`:

```C
p->trapframe->a0 = syscalls[num]();
```
What if I set it back again? Like:

```C
if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    if (num == SYS_sigreturn)
    {
      // restore the value again, see if it works
      p->trapframe->a0 = p->a0;
    }
```

`syscall()` then returns to `usertrap()`. Let's see if there is any other chance that modifies `p->trapframe->a0`.

I thought this is good enough, but sadly sometimes both `p->a0` and `p->trapframe->a0` is not 1337. I'm not 100% sure, but I think the reason is, when we save the value into `p->trapframe->a0`, we don't have the return value from `foo()` yet -- think about it. `a0` contains the return value, IF there is a function call, but if there is no `foo()` call, `a0` could be anything. And then timer interrupt occurs (for consecutively twice), and `periodic()` got called, and in turn `sys_sigreturn()` gets called, and we don't have the proper `a0` value.

OK I talked with ChatGPT and I think the ^ is wrong: if usertrap() occurs in the middle of `foo()` and calls `sig_return()`, it should still return to the original palce and keeps running `foo()`. Damn I'm out of ideas now.

Wait I suddenly realized that I forgot to backup `epc`. Now it's all fine. I ran it a few times and test1 passes. But I need to figure out WHY? Actually, more important, why does `test0()` pass?

OK I think I realized why. `test0()` only tests ONCE, while `test1()` requires me to return to exactly where I left, otherwise `foo()` didn't get the 1337 into `a0`.

#### Trial 3

`test2()` is much easier. I got it right in a couple of tries. Basically I just need to add one more variable `inalarm`. If the process is not in alarm, but alarmticks reaches alarmtriggerticks, `usertraps()` saves the registers and switches `p->trapframe->epc`. If it's already in alarm (`inalarm == 1`), do nothing. Once `slow_handler()` reaches the end and calls `sys_sigreturn()`, clears the `inalarm` variable because we are no longer in alarm.

One issue to note that we should do NOTHING if already in alarm. Do not even save the registers in that case, because `sys_sigreturn()` restores them, which makes the program keep going back to the `sigreturn()` line because the saved `epc` comes from the `sigalarm()` call, which means the `epc` is the next instruction in user land, which is `sigreturn()`. An alternative is to modify `sys_sigreturn()` and make sure it only restores the registers if not in alarm.