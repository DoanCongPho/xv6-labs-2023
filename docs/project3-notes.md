# Project 3: Multithreading — Notes

---

## Kiến thức nền

### Thread là gì?
Một **thread** là một đơn vị thực thi trong chương trình. Nhiều threads chạy trong cùng một process, chia sẻ bộ nhớ (heap, global vars) với nhau.

```
Process
├── Thread 1: đang chạy hàm A
├── Thread 2: đang chạy hàm B
└── Thread 3: đang chạy hàm C
```

### Context Switch là gì?
CPU chỉ chạy 1 thread tại một thời điểm. Để "chạy song song", cần liên tục **lưu trạng thái** thread hiện tại và **khôi phục** thread tiếp theo.

Trạng thái của 1 thread = **các register của CPU** lúc nó đang chạy:
- `ra` — return address (quay về đâu)
- `sp` — stack pointer (đỉnh stack)
- `s0–s11` — callee-save registers (biến tạm)

---

## Part 1: Uthread — User-level Thread Switching

**File:** `user/uthread.c`, `user/uthread_switch.S`

### Mục tiêu
Tự xây dựng scheduler ở user-space (không nhờ kernel), implement context switch giữa các threads.

### Cấu trúc

```c
struct context {
    uint64 ra;
    uint64 sp;
    uint64 s0 .. s11;
};

struct thread {
    char           stack[STACK_SIZE];
    int            state;       // FREE / RUNNING / RUNNABLE
    struct context context;     // saved registers
};
```

### thread_create()

```c
t->context.ra = (uint64)func;              // lần đầu chạy → nhảy vào func
t->context.sp = (uint64)(t->stack + STACK_SIZE); // đỉnh stack (stack grow down)
```

### thread_schedule()

```c
thread_switch((uint64)&t->context, (uint64)&next_thread->context);
```

### thread_switch (RISC-V assembly)

```asm
# a0 = &old->context, a1 = &new->context

# Lưu thread hiện tại
sd ra,   0(a0)
sd sp,   8(a0)
sd s0,  16(a0)
...
sd s11,104(a0)

# Khôi phục thread tiếp theo
ld ra,   0(a1)
ld sp,   8(a1)
ld s0,  16(a1)
...
ld s11,104(a1)

ret   # nhảy đến ra → thread mới tiếp tục
```

### Tại sao chỉ lưu callee-save registers?
Theo RISC-V calling convention:
- **Callee-save** (`ra`, `sp`, `s0–s11`): hàm được gọi phải tự bảo quản → ta phải lưu thủ công
- **Caller-save** (`a0–a7`, `t0–t6`): C compiler đã lưu trên stack trước khi gọi `thread_yield()`

### Luồng chạy

```
Lần đầu:   ra = địa chỉ func → ret → chạy func từ đầu
Lần sau:   ra = địa chỉ trong thread_switch → ret → tiếp tục từ chỗ dừng
```

---

## Part 2: Hash Table + Lock Per Bucket

**File:** `notxv6/ph.c`

### Hash Table hoạt động như thế nào?

```
Bucket 0: [key=5] → [key=10] → NULL
Bucket 1: [key=6] → NULL
Bucket 2: [key=7] → [key=12] → NULL
...
```

Key `k` → bucket `k % NBUCKET`. Mỗi bucket là một linked list.  
`insert()` luôn chèn vào **đầu** linked list.

### Race Condition (vấn đề)

```
Trạng thái ban đầu: table[2] → [key=7] → NULL

Thread A: head_A = table[2]   → [key=7]     ← đọc giá trị cũ
Thread B: head_B = table[2]   → [key=7]     ← cùng đọc giá trị cũ

Thread A: table[2] = {key=12 → [key=7]}
Thread B: table[2] = {key=17 → [key=7]}     ← ghi đè! key=12 biến mất
```

### Giải pháp: Lock Per Bucket

```c
pthread_mutex_t locks[NBUCKET];  // mỗi bucket 1 lock riêng

// main(): khởi tạo
for (int i = 0; i < NBUCKET; i++)
    pthread_mutex_init(&locks[i], NULL);

// put(): chỉ lock bucket cần dùng
void put(int key, int value) {
    int i = key % NBUCKET;
    pthread_mutex_lock(&locks[i]);
    // ... insert ...
    pthread_mutex_unlock(&locks[i]);
}
```

### Tại sao nhanh hơn global lock?

```
Global lock — tuần tự hoàn toàn:
  Thread 0: [put]  chờ   [put]  chờ
  Thread 1:  chờ  [put]  chờ   [put]

Lock per bucket — song song khi khác bucket:
  Thread 0: [bucket 0]  [bucket 2]  [bucket 4]
  Thread 1:  [bucket 1]  [bucket 3]
```

### Kết quả đo được

| | puts/second | keys missing |
|--|--|--|
| `./ph 1` | 66,242 | 0 |
| `./ph 2` | 93,841 | 0 |
| Speedup | **1.42x** ✅ | — |

### Cơ chế test

| Test | Kiểm tra |
|--|--|
| `ph_safe` | Output có `"0 keys missing"` cho tất cả threads |
| `ph_fast` | `puts/second(ph 2) / puts/second(ph 1) >= 1.25` |

---

## Part 3: Barrier

**File:** `notxv6/barrier.c`

### Barrier là gì?
Điểm đồng bộ: tất cả threads phải **chờ nhau** trước khi tiếp tục.

```
Thread 1: làm xong → chờ ──┐
Thread 2: làm xong → chờ ──┤→ tất cả tiếp tục cùng lúc
Thread 3: làm xong → chờ ──┘
```

### Công cụ: Condition Variable

```c
pthread_mutex_t mutex;
pthread_cond_t  cond;

// Ngủ và chờ:
pthread_cond_wait(&cond, &mutex);    // release lock, ngủ

// Đánh thức tất cả:
pthread_cond_broadcast(&cond);
```

### Implement barrier()

```c
void barrier() {
    pthread_mutex_lock(&bstate.barrier_mutex);

    bstate.nthread++;

    if (bstate.nthread == nthread) {
        // Thread cuối cùng đến → đánh thức tất cả
        bstate.round++;
        bstate.nthread = 0;
        pthread_cond_broadcast(&bstate.barrier_cond);
    } else {
        // Chưa đủ → ngủ chờ
        int round = bstate.round;
        while (bstate.round == round)
            pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }

    pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

### Tại sao cần `round`?
Tránh thread nhanh chạy sang round mới, tăng `nthread` trong khi round cũ chưa reset xong.

```
Round 1 xong → round++ = 2, nthread = 0
Thread nhanh vào round 2 → nthread++ = 1
Thread chậm vẫn trong round 1 → while(round == 1) → thoát vì round đã = 2 ✅
```

---

## Tổng kết

| Part | File | Kỹ thuật chính |
|--|--|--|
| 1 | `user/uthread.c`, `user/uthread_switch.S` | Context switch, RISC-V assembly |
| 2 | `notxv6/ph.c` | Mutex lock, lock per bucket |
| 3 | `notxv6/barrier.c` | Condition variable, broadcast |
