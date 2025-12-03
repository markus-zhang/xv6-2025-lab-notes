### Trial 1

I managed to complete the first lab using the hints, but TBH I still feel pretty awkwardly uncomfortable about it.

There are a few discrepancies between what happened in GDB and what is described in the Intel 8254x manual.

Hereby I recorded everything I saw in GDB. I first execute `b e1000_recv` to setup a bp at `e1000_recv()`. And then I switched to a third window and run `python3 nettest.py rxone`. GDB immediately caught the bp. I the  investigated the content of the ring buffer by executing `p/x rx_ring`. I can see that although all 16 descriptors have valid `addr`, only the first one contains any data, because it has `status` as `0x7`.

Checking the manual, `0x7` means that bit2(IXSM - ignore checksum indication), bit1(EOP) and bit0(DD) are all 1. This means that this is the last descriptor for the incoming packet (EOP), and hardware is done with the descriptor (DD).

All ^ makes perfect sense. I then `s` a few steps until the program calls `net_rx()`. Doing this doesn't change anything about the ring buffer, which makes sense. 

Please note that `net_rx()` actually frees the buf at `addr`, so when we call `kalloc()` and give the physciall address to `addr`, it gives exactly the same address, the one just freed. But if we investigate the address by using `x/16c rx_ring[i].addr` then it just shows garbage data.

Eventually it `break` from the while loop and dropped into `devintr()` in `trap.c`. Particularly, it goes into`plic_complete()` with `irq` as 33. I think it just tells the PLIC we have served the IRQ.

Then it immediately jumps into `e1000_recv()` again. This time I can see that the second element of the ring buffer has some new data. And I captured what's in the buffer by looking into the memory address. `x/16c 0x87ed3000` shows a bunch of characters -- 'R', 'T', '\0', '\022', '4', 'V', 'R', 'U', '\n', '\0', '\002', '\b', '\0', 'E'. This looks like garbage but is what I expected. It matches what `tcpdump -XXnr packets.pcap` should produce.

Overall I think it matches what I exepct. However there is one thing that is confusing: I still don't understand how the tail and head pointers work. In the manual it says:

```
Base---->   -----------------
            |xxxxxxxxxxxxxxx|
            -----------------
            |xxxxxxxxxxxxxxx|   
            -----------------
            |               |   <-----Head
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |               |
            -----------------
            |xxxxxxxxxxxxxxx|   <------Tail
            -----------------
            |xxxxxxxxxxxxxxx|
            -----------------
            |xxxxxxxxxxxxxxx|
            -----------------
```

> Shaded boxes in the ^ figure represent descriptors that have stored incoming packets but have not yet been recognized by software.

The problem is, I found that the first "available" descriptor is NOT the tail one, but the (tail + 1) % 16 one. `int i = (e1000_rdt + 1) % RX_RING_SIZE;` this definitely matches what the LAB says but I have no idea why it's the case. It doesn't really match the ^ figure in which Tail points to a shaded box. I can also confirm that looking into GDB, the first packet is stored in descriptor 0, while rdt is 15, and the second packet is stored in descriptor 1, while rdt is 1.


### Trial 2

Damn this must be the most confusing lab I ever wrote. WTH is it talking about?

### Trial 3

OK with some serious discussion with ChatGPT, I finally managed to complete most of the work. `make grade` passes with 160/171 except for the final `free`, which I might take a look later.

Honestly I don't enjoy this lab and I perhaps relied too much on ChatGPT. But still, I do not take any code from ChatGPT so I think it's fine.

As usual, a lot of mistakes are from misunderstanding of the specifications of the lab. e.g. I managed to pass `ping0` and `ping1` without much trouble, but `ping2` took me about at least half a week.

I created a buch of dumping functions to help me pinpoint the issue today, and evertually I found something weird. The `bq` queue for port `2006` and `2007` only has 0 or 1 data when `sys_recv()` is called. This is wrong -- it is supposed to have 5 items in each queue.

The issue is in `e1000_recv()`. TBH I looked at every place except for this one, because it got pass `ping0` and `ping1` without trouble. The hints say: check DD and stop if it's 0. OK so I figured that I should literally break from the loop. Unfortunately I got that one wrong -- I still need to loop through all the buffers.