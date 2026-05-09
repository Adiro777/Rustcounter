# rustcounter — an atomic counter as a Linux kernel module

A small character device in safe Rust that demonstrates Rust's compiler-enforced data-race prevention at the kernel level.

- `write()` to `/dev/rustcounter` → increments the counter
- `read()` from `/dev/rustcounter` → returns the current value as ASCII
- Two processes writing in parallel always produce the exact total. No locks, no lost increments

## Demo

Basic usage:

```text
$ sudo insmod rustcounter.ko
$ ls -la /dev/rustcounter
crw------- 1 root root 10, 263 May  5 19:47 /dev/rustcounter
$ sudo cat /dev/rustcounter
0
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ echo bump | sudo tee /dev/rustcounter > /dev/null
$ sudo cat /dev/rustcounter
3
$ sudo dmesg | tail -3
rustcounter: incremented to 1
rustcounter: incremented to 2
rustcounter: incremented to 3
```

Concurrent writes from two processes:

```text
# Terminal A:
$ for i in {1..1000}; do echo bump | sudo tee /dev/rustcounter > /dev/null; done

# Terminal B (started while A is still running):
$ for i in {1..1000}; do echo bump | sudo tee /dev/rustcounter > /dev/null; done

# After both finish:
$ sudo cat /dev/rustcounter
2000
```

Exactly 2000, every time. Equivalent C with `count++` and no lock would silently lose increments. Rust's type system refuses to compile that pattern.

## What this is

- Out-of-tree Linux kernel module written in Rust using [Rust-for-Linux](https://rust-for-linux.com/) APIs
- State: one `AtomicU64` (the counter) + one `AtomicBool` (a flag for clean `cat` semantics)
- No `Mutex`, no `RwLock`, no spinlock. Atomics only
- `fetch_add` compiles to a single `LDADDAL` instruction on aarch64 (`LOCK XADD` on x86)

## Build & run

**Prerequisites:** A kernel built with `CONFIG_RUST=y` and matching `rustc`. Ubuntu 26.04 LTS in a Multipass VM provides both.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) kmod
sudo apt install -y rustc-1.93 rust-1.93-src bindgen
sudo update-alternatives --install /usr/bin/rustc rustc /usr/bin/rustc-1.93 100

git clone https://github.com/Adiro777/Adi_Roitburg_OS_HW5
cd Adi_Roitburg_OS_HW5
make
sudo insmod rustcounter.ko
```

**Use:**

```bash
sudo cat /dev/rustcounter                          # read current value
echo bump | sudo tee /dev/rustcounter > /dev/null  # increment
sudo rmmod rustcounter                             # unload
```

## Code tour

- **`init`** — registers the misc device, prints the load message
- **Static state:**
```rust
  static COUNT: AtomicU64 = AtomicU64::new(0);
  static CONSUMED: AtomicBool = AtomicBool::new(false);
```
- **`write_iter`** — the interesting line:
```rust
  let new_count = COUNT.fetch_add(1, Ordering::SeqCst) + 1;
```
  Drains user input but ignores it; only the act of writing matters.
- **`read_iter`** — atomically reads the counter, formats with `CString::try_from_fmt`, copies to user space via `iov.simple_read_from_buffer` (safe wrapper around `copy_to_user`)

## Design notes

- **Atomic over mutex:** A `Mutex<u64>` would work but adds memory fences and potential scheduler interaction. `AtomicU64::fetch_add` is one CPU instruction, cheapest correct synchronization.
- **`SeqCst` ordering:** `Relaxed` would technically work for a pure counter. `SeqCst` is the safe default; pick it unless you can articulate why weaker is correct, especially in kernel code.
- **`CONSUMED` flag:** Without it, `cat` loops forever. Non-zero `read()` tells `cat` "call me again." The flag lets the first read after each write return the value, then signals EOF until the next write.

## Future work

- `/proc/rustcounter` entry for non-consuming reads
- `RESET` command — writing `RESET\n` zeroes the counter instead of incrementing
- Per-process counts via `HashMap<pid, u64>`
- Histogram of write rates using jiffies-based timestamps
- Saturating semantics (clamp at `u64::MAX` instead of wrapping)

## License

GPL-2.0, to match the Linux kernel.

## Notes

First kernel module, built for CMSI 3510 (Operating Systems) at LMU. Targets Ubuntu 26.04 LTS's stock kernel on Multipass + Apple Silicon.
