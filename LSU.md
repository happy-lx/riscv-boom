https://docs.boom-core.org/en/latest/sections/load-store-unit.html?highlight=lsu



![截屏2021-09-17 下午4.16.07](/Users/lixin/Desktop/截屏2021-09-17 下午4.16.07.png)

## Store Instructions

Entries in the Store Queue are allocated in the *Decode* stage ( stq(i).valid is set). A “valid” bit denotes when an entry in the STQ holds a valid address and valid data (stq(i).bits.addr.valid and stq(i).bits.data.valid). Once a store instruction is committed, the corresponding entry in the Store Queue is marked as committed. The store is then free to be fired to the memory system at its convenience. **<u>Stores are fired to the memory in program order</u>.**

### Store Micro-Ops

Stores are inserted into the issue window as a single instruction (as opposed to being broken up into separate addr-gen and data-gen [UOP](https://docs.boom-core.org/en/latest/sections/terminology.html#term-micro-op-uop) s). This prevents wasteful usage of the expensive issue window entries and extra contention on the issue ports to the **LSU**. A store in which both operands are ready can be issued to the **LSU** as a single [UOP](https://docs.boom-core.org/en/latest/sections/terminology.html#term-micro-op-uop) which provides both the address and the data to the LSU. While this requires store instructions to have access to two register file read ports, this is motivated by a desire to not cut performance in half on store-heavy code. Sequences involving stores to the stack should operate at IPC=1!

However, it is common for store addresses to be known well in advance of the store data. **<u>Store addresses should be moved to the STQ as soon as possible to allow later loads to avoid any memory ordering failures.</u>** Thus, the issue window will emit uopSTA or uopSTD [UOP](https://docs.boom-core.org/en/latest/sections/terminology.html#term-micro-op-uop) s as required, but retain the remaining half of the store until the second operand is ready.

## Load Instructions

Entries in the Load Queue (LDQ) are allocated in the *Decode* stage (`ldq(i).valid`). In **Decode**, **<u>each load entry is also given a *store mask* (`ldq(i).bits.st\_dep\_mask`)(这个应该是load-wait table之类的推测的)</u>**, which marks which stores in the Store Queue the given load depends on. When a store is fired to memory and leaves the Store Queue, the appropriate bit in the *store mask* is cleared.**<u>load需要等到所有的相关的store发射后才能发射，应该只要数据准备好了就可以从保留站出来放到LQ里面，让LQ去控制需不需要发射</u>**

Once a load address has been computed and placed in the LDQ, the corresponding *valid* bit is set (`ldq(i).addr.valid`).

Loads are optimistically fired to memory on arrival to the **LSU** (getting loads fired early is a huge benefit of out–of–order pipelines). Simultaneously, **<u>the load instruction compares its address with all of the store addresses that it depends on. If there is a match, the memory request is killed(memory request是指发到访存子系统的请求)</u>**. **<u>If the corresponding store data is present, then the store data is *forwarded* to the load and the load marks itself as having *succeeded*.</u>** **<u>If the store data is not present, then the load goes to *sleep*. Loads that have been put to sleep are retried at a later time.</u>** [[1\]](https://docs.boom-core.org/en/latest/sections/load-store-unit.html?highlight=lsu#id3)


发生了load-store违例的load是需要被kill掉的，不要让他去访存。
如果load相关的store的数据在，直接给load，如果不在，load需要等一会再送出去。
如果load去检查store的时候此时store的地址没有就绪怎么办？

任何一个ls流水线的请求都有可能发生TLB miss，它们在IQ里面的项目应该发射出来之后就清空掉了，
但它们有信息在lq和sq里面，所以一旦发生TLB miss，可以在lq和sq里面去重发，应该就是下面这个retry的含义
wakeup指的是load检查的时候发生了违例(或者前面还有store地址不就绪？)，但是store的数据还没就位，所以需要在lq里面sleep


load queue的retry是地址valid，并且是虚地址，并且没有block
store queue的retry是地址valid，并且是虚地址
load queue的wake up是地址valid，并且实地址，没有block，没有execute没有成功


这个IQ会发带地址和数据的store请求，只带地址的store请求，只带数据的store请求(为什么？)
因为IQ中的store指令只要它的地址源操作数就绪了就可以发只带地址的store请求，防止在这个
store后面的load无法检查相关性，导致降低并发度，但是IQ将其发射出去之后不会释放这个entry，
等它的data ready了就再发只带数据的store请求，这时就可以清除掉这个entry了，非常精妙


### 总体框架
+ 在deocde阶段，接受所有的load store指令到lq和sq中
+ load store指令会进入到IQ中等待源操作数ready发射
+ load只有一个源寄存器，准备好了就可以发射，store的情况见上面的分析
+ load和store分别到对应的流水线去，经过地址计算单元得到虚地址，然后作为一个请求给lq和sq
+ lq和sq就会分别监听对应的流水线
    + 如果有load发过来，就把它的虚拟地址写到lq中，设置为valid
    + 如果有store的地址过来，就把它的虚拟地址写到sq中，设置为valid
    + 如果有store的数据过来，就把它的数据写到sq中，设置为valid
+ 因为从IQ中把东西发过来(地址，数据)，记录到了lq和sq中其实就ok了，它有没有去后面的访存流水级其实不重要，因为lq和sq中有它，可以从中发
+ 每一条流水级往后面发什么是有优先级的，如果这个


