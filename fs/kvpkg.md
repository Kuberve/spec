# Kuberve Package Format (`.kvpkg`)

## Общее

`.kvpkg` = **tar архив** (ustar, не pax, без сжатия). 

**Почему tar:**
- Host-friendly: `tar cf hello.tar hello/` работает в любом Linux/Mac
- Отладка: `tar tvf hello.tar` показывает структуру
- Стандарт: совместим с `python3 -m tarfile`, `bsdtar`, `GNU tar`
- Простой parser: фиксированный 512-байтный header, ~150 строк Rust

**Почему не gzip (пока):**
- Сжатие в 3-4 раза нужно когда скачиваем по сети
- Без сети (локально с диска) размер не критичен
- gzip добавим в B.7 (HTTP-репозиторий)

## Расширение

`.kvpkg` — метка для менеджера пакетов и пользователя. Внутри — обычный tar.

## Структура архива

```
manifest.txt          ← обязателен, в корне архива
bin/<name>            ← ELF, обязательно для executable пакета
lib/<name>.so         ← опционально
conf/<name>.cfg       ← опционально
data/<...>            ← опционально (иконки, ресурсы)
```

**Правила:**
- `manifest.txt` — первая запись (для быстрого чтения без полного парсинга)
- Все пути **относительные**, без ведущего `/`
- Директории — с trailing `/` (tar convention)
- Порядок остальных файлов — как в `tar cf` (обычно алфавитный, но не обязательно)

## manifest.txt

Line-based, key=value:

```
name=hello
version=1.0.0
description=Hello world example
priority=2
bin=hello
```

**Обязательные поля:**
- `name` — имя пакета (8.3-safe, ≤ 8 символов, без `.`)
- `version` — semver (major.minor.patch)
- `bin` — имя бинаря в `bin/` (без `.elf`)

**Опциональные:**
- `description` — однострочное описание
- `priority` — 0..255 для `spawn` (по умолчанию 2)

**Парсер:** `lines() → trim() → skip # → split('=')`. Как в `init.cfg`.

## Установка

`kv install <path.kvpkg>`:

1. `sys_open(path, O_RDONLY)` → fd
2. Прочитать `manifest.txt` (первая запись) → parse
3. `sys_mkdir("/apps/<name>")` (idempotent)
4. Для каждой записи tar:
   - `typeflag='5'` (dir) → `sys_mkdir("/apps/<name>/<path>")`
   - `typeflag='0'` (file) → `sys_mkdir(parent)` + `sys_open(O_CREAT)` + `sys_write_file` + `sys_close`
5. Скопировать манифест в `/apps/<name>/manifest.txt` (для `kv list`)
6. Опционально (Phase 2.3): скопировать `.kvpkg` в `/packages/<name>.kvpkg` для future uninstall

**Идемпотентность:** повторная установка — перезапись (не ошибка). `sys_mkdir` уже idempotent, `sys_open(O_CREAT)` на существующий файл — открывает, `sys_write_file` пишет с offset 0.

## Удаление

`kv remove <name>`:

1. `sys_readdir("/apps/<name>")` — список файлов
2. Рекурсивный обход:
   - Файл → `sys_unlink`
   - Директория → сначала рекурсивно, потом `sys_rmdir`
3. `sys_rmdir("/apps/<name>")`

**Ограничение:** `sys_rmdir` требует пустую директорию. Значит удаление — **снизу вверх** (post-order traversal).

## Список

`kv list`:

1. `sys_readdir("/apps")` → entries
2. Для каждой директории: `sys_open("/apps/<name>/manifest.txt")` + `sys_read` → parse → печатать `name version description`

Если `manifest.txt` нет — печатать `<name> (no manifest)`.

## ustar header — полный формат

512 байт на запись:

| offset | size | поле | формат |
|---|---|---|---|
| 0 | 100 | name | null-terminated ASCII |
| 100 | 8 | mode | octal, обычно "0000644\0" |
| 108 | 8 | uid | octal, "0000000\0" |
| 116 | 8 | gid | octal, "0000000\0" |
| 124 | 12 | size | octal, размер данных (для dir = 0) |
| 136 | 12 | mtime | octal, unix timestamp |
| 148 | 8 | checksum | 6 octal + "\0 " (space) |
| 156 | 1 | typeflag | '0'=file, '5'=dir, '\0'=file |
| 157 | 100 | linkname | null-terminated (для symlink, не используем) |
| 257 | 6 | magic | "ustar\0" |
| 263 | 2 | version | "00" |
| 265 | 32 | uname | null-terminated |
| 297 | 32 | gname | null-terminated |
| 329 | 8 | devmajor | octal (для device, не используем) |
| 337 | 8 | devminor | octal |
| 345 | 155 | prefix | null-terminated (для длинных путей) |
| 500 | 12 | padding | null |

Данные идут сразу после header, размер = `size` из header, padding до 512 байт нулями.

**Конец архива:** два блока по 512 нулей.

**Checksum:** сумма всех байт header (при вычислении checksum-поле = 8 пробелов).

## Парсер (псевдокод)

```
loop {
    header = read_512(fd)
    if header == all_zeros: break   // конец архива
    name = parse_str(header[0..100])
    size = parse_octal(header[124..136])
    typeflag = header[156]
    
    if typeflag == '5':   // dir
        sys_mkdir(dest + name)
    else if typeflag == '0' or typeflag == 0:   // file
        parent = dirname(dest + name)
        sys_mkdir(parent)
        fd_out = sys_open(dest + name, O_CREAT)
        remaining = size
        while remaining > 0:
            chunk = read(min(512, remaining))
            sys_write_file(fd_out, chunk)
            remaining -= len(chunk)
        sys_close(fd_out)
        skip padding (to 512 boundary)
}
```

**`parse_octal`:** bytes до `\0` или space, `to_digit(8)`.

## Что не делаем (MVP)

- Symlinks, hardlinks — `typeflag='1'/'2'`, не поддерживаем
- Long names >100 символов (pax `././@PaxHeader`) — отказ
- Sparse files — отказ
- Досье подписей (Ed25519) — Phase 2.4, после репозитория
- Compression (gzip) — Phase 2.3, с HTTP
- Dependencies (`dep=foo>=1.0`) — Phase 2.4

## Пример

```bash
# На хосте
mkdir hello
echo "name=hello
version=1.0.0
description=Hello world
bin=hello" > hello/manifest.txt
mkdir hello/bin
cp user/target/.../hello hello/bin/hello
tar cf hello.kvpkg -C hello manifest.txt bin/hello
mcopy -o -i nvme.img hello.kvpkg ::/tmp/hello.kvpkg

# В Kuberve
kv install /tmp/hello.kvpkg
kv list
# hello  1.0.0  Hello world
```

## FAT32 8.3 ограничения

Все имена в `/apps/<name>/` — 8.3:
- `manifest.txt` → `MANIFEST TXT` (10 символов в 11 байт, ок)
- `bin/hello` → `HELLO` (без ext, ок)
- `bin/long_name` → `LONG_NA~1` (LFN, наш `find_entry` **не найдёт**)

**Правило:** имена файлов ≤ 8 символов без ext, или ≤ 8+3 с ext. Нарушение — файл **установится** (через `sys_write_file`), но `kv list`/`kv remove` его не найдут. На KuberveFS (Phase 3) — любой длины.

## Version history

| Версия | Что |
|---|---|
| v0.14.4 (B.1) | Спека, tar ustar, manifest.txt, без сжатия |
| Phase 2.3 | gzip, HTTP, репозиторий |
| Phase 2.4 | Ed25519 подписи, dependencies |
| Phase 3 | KuberveFS, любой длины имена |
