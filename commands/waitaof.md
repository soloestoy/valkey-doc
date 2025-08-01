This command blocks the current client until all previous write commands by that client are acknowledged as having been fsynced to the AOF of the local Valkey and/or at least the specified number of replicas.

`numlocal` represents the number of local fsyncs required to be confirmed before proceeding.
When `numlocal` is set to 1, the command blocks until the data written to the Valkey instance is confirmed to be persisted to the local AOF file.
The value 0 disables this check.

If the timeout, specified in milliseconds, is reached, the command returns even if the specified number of acknowledgments has not been met.

The command **will always return** the number of primaries and replicas that have fsynced all write commands sent by the current client before the `WAITAOF` command, both in the case where the specified thresholds were met, and when the timeout is reached.

A few remarks:

1. When `WAITAOF` returns, all the previous write commands sent in the context of the current connection are guaranteed to be fsynced to the AOF of at least the number of primaries and replicas returned by `WAITAOF`.
2. If the command is sent as part of a `MULTI` transaction (or any other context that does not allow blocking, such as inside scripts), the command does not block but instead returns immediately the number of primaries and replicas that fsynced all previous write commands.
3. A timeout of 0 means to block forever.
4. Since `WAITAOF` returns the number of fsyncs completed both in case of success and timeout, the client should check that the returned values are equal or greater than the persistence level required.
5. `WAITAOF` cannot be used on replica instances, and the `numlocal` argument cannot be non-zero if the local Valkey does not have AOF enabled.

Limitations
---
It is possible to write a module or Lua script that propagate writes to the AOF but not the replication stream.
(For modules, this is done using the `fmt` argument to `RedisModule_Call` or `RedisModule_Replicate`; For Lua scripts, this is achieved using `server.set_repl`.)

These features are incompatible with the `WAITAOF` command as it is currently implemented, and using them in combination may result in incorrect behavior.

Consistency and WAITAOF
---

Note that, similarly to [`WAIT`](wait.md), `WAITAOF` does not make Valkey a strongly-consistent store.
Unless waiting for all members of a cluster to fsync writes to disk, data can still be lost during a failover or a Valkey restart.
However, `WAITAOF` does improve real-world data safety.

Implementation details
---

Valkey tracks and increments the replication offset even when no replicas are configured (as long as AOF exists).

In addition, Valkey replicas asynchronously ping their primary with two replication offsets: the offset they have processed in the replication stream, and the offset they have fsynced to their AOF.

Valkey remembers, for each client, the replication offset of the produced replication stream when the last write command was executed in the context of that client.
When `WAITAOF` is called, Valkey checks if the local Valkey and/or the specified number of replicas have confirmed fsyncing this offset or a greater one to their AOF.

## Examples

```
> SET foo bar
OK
> WAITAOF 1 0 0
1) (integer) 1
2) (integer) 0
> WAITAOF 0 1 1000
1) (integer) 1
2) (integer) 0
```

In the above example, the first call to `WAITAOF` does not use a timeout and asks for the write to be fsynced to the local Valkey only; it returns with [1, 0] when this is completed.

In the second attempt we instead specify a timeout, and ask for the write to be confirmed as fsynced by a single replica.
Since there are no connected replicas, the `WAITAOF` command unblocks after one second and again returns [1, 0], indicating the write has been fsynced on the local Valkey but no replicas.
