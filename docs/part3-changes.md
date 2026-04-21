# Part 3: Barrier — Changes to `notxv6/barrier.c`

## What was changed

### Implemented `barrier()`

```c
// Before:
static void barrier() {
    // YOUR CODE HERE
}

// After:
static void barrier() {
    pthread_mutex_lock(&bstate.barrier_mutex);

    bstate.nthread++;

    if (bstate.nthread == nthread) {
        // Last thread to arrive — wake everyone up
        bstate.round++;
        bstate.nthread = 0;
        pthread_cond_broadcast(&bstate.barrier_cond);
    } else {
        // Not enough threads yet — sleep and wait
        int r = bstate.round;
        while (bstate.round == r)
            pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }

    pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

---

## How it works

```
Thread arrives at barrier():
  nthread++

  if nthread == total threads:
    round++          ← advance to next round
    nthread = 0      ← reset for next round
    broadcast        ← wake up all sleeping threads
  else:
    save current round into local r
    sleep until bstate.round != r
```

---

## Why save `round` into a local variable?

Without it, a fast thread could race back and increment `nthread` before
a slow thread wakes up, causing the slow thread to sleep forever (deadlock).

```
Round 1 ends → round=2, nthread=0, broadcast

Thread 0 (fast): exits barrier → loops back → nthread++ = 1

Thread 1 (slow): just woke up → checks exit condition
  if while(nthread > 0)  → sees nthread=1 → sleeps again → DEADLOCK ❌
  if while(round == r)   → r=1, bstate.round=2 → exits correctly ✅
```

---

## Test results

```bash
./barrier 1  → OK; passed ✅
./barrier 2  → OK; passed ✅
./barrier 4  → OK; passed ✅
```
