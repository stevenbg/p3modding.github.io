# Multiplayer
P3 runs multiplayer over two threads. The **game thread** is the one that ticks the world
and executes [operations](./operations.md); the **network thread** runs every network
operation, receiving from and sending to the other clients.

The two meet at the operation queues, and one client is the **host**: it chooses the order
in which operations are executed, which is what keeps the clients' worlds identical. See
[Operation Synchronization](./multiplayer/operation-synchronization.md) for the path a
single operation takes through the queues.

The shared queues are accessed from both threads **without correct locking**, which is a
defect rather than a design - it can make operations disappear and desync the session. It is
written up as its own bug: [Multiplayer Locks](./bugs/multiplayer-locks.md).
