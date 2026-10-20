# IPC Message Format

## Layout

```rust
#[repr(C)]
pub struct Message {
    pub tag: u64,        // тип сообщения
    pub data: [u64; 7],  // payload, 56 байт
}
```

**Размер:** 64 байта (8 + 56).
**Выравнивание:** 8 байт (естественное для u64).

## Зачем 64 байта

- Половина/целая cache line (зависит от CPU).
- Копируется за несколько инструкций.
- Влезает в регистр-пару `stp/ldp` при сохранении контекста.
- Аналог: QNX (64 байта), seL4 (64–128 байт), MINIX 3 (64 байта).

Большие данные — через shared memory (v0.7+). 64 байта — для сигналов и маленьких запросов.

## Теги

Расширяемый namespace. Тег определяет, как интерпретировать `data[]`.

| Тег | Имя | data[0] | data[1] | data[2..7] |
|---|---|---|---|---|
| 1 | WRITE | ptr | len | — |
| 2 | IRQ | intid | — | — |

## Пример: WRITE

Клиент пишет в uart_server:
```rust
let text = b"hello\n";
let msg = Message {
    tag: 1,  // WRITE
    data: [text.as_ptr() as u64, text.len() as u64, 0, 0, 0, 0, 0],
};
sys_send(uart_pid, &msg);
```

Uart_server получает, читает `data[0]` как `*const u8`, `data[1]` как `usize`, пишет в UART.

**Важно (до v0.7):** `ptr` указывает в **общее** адресное пространство. Uart_server видит ту же память, что client (identity mapping). После v0.7 (per-process AS) это сломается — придётся копировать данные **в** `Message.data` (если 56 байт хватает) или вводить shared memory.

## Пример: IRQ

Kernel при IRQ 33 шлёт uart_server:
```rust
let msg = Message {
    tag: 2,  // IRQ
    data: [33, 0, 0, 0, 0, 0, 0],
};
ipc_send(2, msg);
```

Uart_server читает `data[0]` как intid, обрабатывает, зовёт `sys_irq_ack(33)`.

## Правила для новых тегов

1. Один тег = одно сообщение. Не мультиплексировать (`tag=1, subtype=2`) — заводи новый тег.
2. Все неиспользованные `data[]` = 0 (детерминизм, отладка).
3. Указатели — только в `data[0..1]`, остальное — значения. Легче отлаживать.
```

## `spec/memory/user-layout.md`

# User Memory Layout (ARM64)

## Регионы

| Регион | VA | Размер | AP | UXN | Использование |
|---|---|---|---|---|---|
| User code | 0x40400000 | 2MB | 01 | 0 | ELF PT_LOAD (RX для EL0) |
| User data/stack | 0x40600000 | 2MB | 01 | 1 | Стеки, глобальные данные |
| User heap | 0x40800000 | 2MB | 01 | 1 | Bump allocator |

AP (Access Permission):
- `00` — EL1 только (kernel)
- `01` — EL1 + EL0 (user)

UXN:
- `0` — исполняемо с EL0
- `1` — не исполняемо с EL0

## Стеки

Каждый EL0-поток получает свой стек в регионе 0x40600000–0x407FFFFF.
Размер стека — 16 KiB, растёт вниз.

Первый стек: `0x40700000`.
Второй: `0x40780000`.
Третий: `0x407C0000`.
...

## Heap

`0x40800000–0x409FFFFF` (2MB). Bump allocator в `kuberve-libc`.
`HEAP_START = 0x40800000`, `HEAP_END = 0x40A00000` (exclusive).

После v0.7 (per-process AS) — heap **на процесс**, через `sys_sbrk` или `sys_mmap`.

## Kernel-регионы (для справки)

| Регион | VA | AP |
|---|---|---|
| Device | 0x00000000–0x3FFFFFFF | 00 (кроме UART @ 0x09000000) |
| Kernel code | 0x40000000 | 00 |
| Kernel data | 0x40200000 | 00 |

**EL0 не может** читать/писать kernel-регионы (AP=00). При попытке — `Data abort` (EC=0x24 или 0x25).

## Переход к per-process AS (v0.7)

Сейчас **все EL0-потоки видят одну и ту же память** (identity mapping, один `TTBR0`).

В v0.7:
- Каждый процесс = свой `TTBR0` + свои page tables.
- `0x40400000` у процесса A — физически **другая** страница, чем у B.
- Kernel копирует данные между процессами через `copy_to_user`/`copy_from_user`.
- IPC с указателями (`data[0] = ptr`) **сломается** — нужно копировать данные в `Message` или shared memory.