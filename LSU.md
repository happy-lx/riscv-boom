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

