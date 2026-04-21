# Part 2: Using Threads — Changes to `notxv6/ph.c`

## What was changed

### 1. Added per-bucket locks (global declaration)

```c
// Added after line: int nthread = 1;
pthread_mutex_t locks[NBUCKET];
```

One mutex per bucket instead of a single global lock, so threads operating on different buckets can run in parallel.

---

### 2. Initialized locks in `main()`

```c
// Added after: nthread = atoi(argv[1]);
for (int i = 0; i < NBUCKET; i++)
    pthread_mutex_init(&locks[i], NULL);
```

---

### 3. Lock/unlock the correct bucket in `put()`

```c
// Before:
static void put(int key, int value) {
    int i = key % NBUCKET;
    // no lock — race condition here
    ...
}

// After:
static void put(int key, int value) {
    int i = key % NBUCKET;
    pthread_mutex_lock(&locks[i]);     // lock only bucket i
    ...
    pthread_mutex_unlock(&locks[i]);   // unlock bucket i
}
```

---

## Why lock per bucket (not a global lock)?

| | Safety | Speed |
|--|--|--|
| No lock | ❌ keys missing | Fast but wrong |
| 1 global lock | ✅ | Same as single thread |
| Lock per bucket | ✅ | ~1.4x faster with 2 threads |

Two threads inserting into **different buckets** hold different locks → they run **fully in parallel**.  
Two threads inserting into the **same bucket** → one waits → correct serialization.

---

## Test results

```
./ph 1 → 66,242 puts/second, 0 keys missing
./ph 2 → 93,841 puts/second, 0 keys missing (speedup: 1.42x)
```

- `ph_safe` ✅ — 0 keys missing with 2 threads
- `ph_fast` ✅ — speedup >= 1.25x requirement
