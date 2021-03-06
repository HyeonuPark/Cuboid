Cuboid architecture
===================

Cuboid is designed with parallelism in mind. Each cuboid agent must be able to read/write data in anytime, while transferring them among agents can be occurred eventually. Garbage collection is running concurrently in separated agent, and it must not change the ordering of objects.

A working cuboid implementation is consist of a group of shared memory and some JS objects that handle this memory, and some data written on it. Let's call them as `context`, `agent`s, and `record`s. Each agents may be running in parallel, within separated OS threads and JS context like WebWorker.

Each context has single master agent and zero or more normal agents. All agents including master can read/write record anytime. The only difference is that master agent performs garbage collection. For users, creating agent without existing context will make master agent with newly generated context.

Each cuboid context has its own 4 byte namespace for record. This memory space consists of a list of pages, a unit of allocation whose size is generally 4kb. Each page is allocated when needed and re-used when possible. Pages construct chapters, which can be used as an arbitrary sized linear memory block. Chapter is the logical unit of cuboid context.

Each context usually has 2 chapters: newbie and oldbie chapter. Each chapter is garbage collected separately. Newly generated records are always written to newbie chapter. If this record is survived after 2 garbage collection, it will moved to oldbie chapter. This 2-level GC is based on common heuristics that most objects are live short, while others are tends to live long.

All records have header fields that contains reference count, new address after GC, and struct id to indicate what type this record is. Struct definition contains informations about each field of records. It should be provided before the context is initialized, and must be identical among agents that shares this context.

As not all data can be presented in fixed size, cuboid provides blob chapter. Blob chapter's pages may not has fixed size, as blobs can be larger then 4kb. They share lifetime with matching blob record, which is a normal record lives in normal chapter that represent this blob.

To provide deterministic comparability, every pair of records should keeps same order even after garbage collection. But GC shouldn't block other agents to perform read/write. These problems sound hard to solve. But, as records are immutable, all records in single cuboid context forms linear history and all references has same direction. This is the essential assumption of cuboid lifetime and garbage collection.

When agent attempts to write some record, it first check whether the context is currently `commitable`. If commitable, it atomically allocate chapter space and write record to chapter. If not, it just store data to agent-local queue. This queue is emptied when the context become commitable and all deferred records are written in order.

When agent attempts to read some record, it first check this record is whether in shared context or in local queue. If it's in local, it just return locally stored data. If it's in shared context, the agent create local handle object that represent this record, and return the data. Read operation can be cached to handle.

When master agent decide to start garbage collection, it first set its context as `non-commitable` and tell it to other agents with same context. When all agents agreed with it, master agent allocate new chapter and copy living records to it in `oldest-first` order, and fill old record's `new address` field. During this, old chapter is still accessible as read-only. And writing is still possible to locally-accessible queue, with same order as creation time. After copy, master agent set context's state as commitable and tell it to other agents so they can change local handles to new address. And clear old chapter after all agents confirmed to moved to new chapter.

It's guaranteed to not change any order among two GC-ed records, as history of records is linear so when we copy an record, any other living records older then this one are already copied and has new address. This also applied to records generated during GC, as non-committed records cannot be referenced from other agent as they aren't `visible` to them, so maintaining agent-wide ordering is enough for it. And of course all committed records are definitely older then non-committed records. So ordering of records keeps persistent during garbage collection.

As cuboid uses reference counted garbage collection, it's important to know when the record become non-visible. Cuboid uses promise-based transaction for it. Every read/write operation using cuboid must be inside some transaction, and it will fail when bound transaction is expired. Transaction is treated as root reference of mark and sweep GC. Any record that cannot be visible from any living transaction is treated as `dead` and can be garbage collected. Due to its immutability and linear history, cuboid can avoid cyclic reference and layered GC at almost zero cost.

Transaction can be created in 2 way: by manually or by listening event from agent. To manually create transaction, you can call agent's method with function that returns promise. This transaction will be expired when returned promise is fulfilled, so it's recommended to pass async function to it. To listen event from agent, just call another method like manual creation, but with one more argument: event name. And agent will automatically create transaction and call your function whenever matching event is received.

Transaction is started with single record from received message. It will be null if you manually created this transaction. You can create as many records while this transaction, with/without some existing transaction. Records you created and from message is treated as `referenced` from transaction, so they will not be GC-ed until this transaction is expired. You can emit some `event` with existing record while transaction. The event also hold reference for its message until every registered listeners start transaction. Note that emitting actually happen after the message is committed. If not, it will be deferred.

To avoid layered GC problem, expiring transaction de-reference every bounded records, recursively. It means when it's detected that de-reference makes record dead, it also de-reference records referenced from died record. This process ensures that clearing some record will not make other records dead, so never let those dead records alive until next GC. Of course this de-reference can be deferred while context is non-commitable.
