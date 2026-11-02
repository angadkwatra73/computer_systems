1. Why introduce virtual memory?
- allwows more space than RAM
- also offers protection from other processes, and allow for multiprogramming
- else each process would have to know about the others. ( connected polygon vs processes -> OS(singular) -> resorces
- don't have have to load the whoel program?

2. Mapping virtual mem -> phy addr
- introduce ideas of making pages
- reducing page table size for cache frienlly lookup

## Question:so whre do the page directories reside?
their entries contain what? - phy addr of the subsequent page tables?

Paging is split into 3 policy questions
- when to fetch (demand or pre paging)
- where to place ( any free frame)
- what to evict ( page replacment)

3. Traversing through pages
- i request a transation
- yes so x86 the regsiter hold physical address of PGD
- kernel level strucutes therefore they are trusted by the hardware
- subsequently we get the physical address of the next table and so on - until the final one

## explanation: . what's the difference between a normal load,store, fetch vs a page-table walk
- when code acceesses adresses they're checked but for walks they are not cehcked for permissoin

-- esentailyl hwo smaller pages help is you need ot load less to acerss an address

##  Idea on how segmentation works in sync w SEG
i------ finally only okay we hevt he idea of paging -- now impelemt--


5.  now we imporove performace w a tLB and demand pagin
  - and also demand paging. - wasn't this always the idea?
- paging is a mechanism, mem in fixed size. by itself it solces external fragemntation
- demand paging is a polocy about when to loads pages - load via page fauultA
- demand paging makies oversubsciption cheap, 


6. Page fault - major, minor - we studies minor earlier

covered demand paging

7. When phy mem runs out - page eviction.replacemnt



Fun question is
- paging is a software thign, levels, heirarcy- OS
- the MMU and TLB is hardware
- do linux/ microsoft for windows talk to these hardware vendors - intel, etc?

