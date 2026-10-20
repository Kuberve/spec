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