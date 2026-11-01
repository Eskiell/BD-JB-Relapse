# BD-JB-Relapse

A **BD-J** (Blu-ray Disc Java) port of the **relapse** PS5 kernel exploit, loaded
through **BD-JB5**. It gains kernel read/write and launches an ELF loader on the
console entirely from Java running inside the Disc Player's BD-J VM — no WebKit entry
point required.

**Status:** work in progress, but functional — it leaks the kernel base, gains kernel
R/W, escalates privileges (uid 0, out of the sandbox) and starts the elfldr payload
server.

> BD-J port by **EchoStretch, zecoxao and Claude**.
> Original *relapse* exploit from the PS5 research community (WebKit PoC).

## Supported firmware

PS5 firmwares up to and including **13.42**. Per-firmware offset tables are included
for **7.00 – 13.60** (`RelapseOffsets.java`); development and testing were done on a
13.42 devkit.

## How it works

Relapse abuses the `aio_multi_wait` **mode-0 waiter use-after-free** (syscall `663` /
`0x297` with `mode = 0`). The dangling waiter is reclaimed to corrupt a `sysctl` OID,
yielding a slow kernel read/write primitive that is then upgraded to a fast
pipe-based R/W. After privilege escalation it hands control to an embedded loader
chain:

- **kexp** (ufm42) — applies the kernel patches and bootstraps the payload loader.
- **elfldr** (ps5-payload-dev) — migrates out to a system service and serves ELF
  payloads on **port 9021**.

## Implementation notes (BD-J port)

Porting the original WebKit/JavaScript exploit to the BD-J VM required working around
several platform constraints on recent firmware:

- **Execute-only `libkernel` `.text`.** Code pages can't be read, so stub scanning
  and ROP gadget hunting are impossible — syscalls are reached through exported
  `libkernel` wrappers instead.
- **The UAF needs `mode = 0`, which the wrappers reject.** `sceKernelAioWaitRequests`
  validates its arguments and refuses `mode = 0` (`EINVAL`), but that mode is exactly
  what triggers the waiter UAF. The raw syscall stubs are not exported, yet they
  physically exist inside **`libkernel_sys`**; the port locates them at runtime by
  computing fixed offsets from a `dlsym`'d anchor and calls them directly with
  `mode = 0`.
- **No generic `syscall()` and no writable JIT before escalation.** `pipe2` is rebuilt
  from `pipe()` + `fcntl`, and the KASLR source address is obtained via `connect()` +
  `getsockname` rather than the (unreachable) interface-list APIs.
- **Clean teardown.** After launching the payload server the exploit restores the
  corrupted `sysctl` OIDs, disarms the crossed pipes and releases the AIO worker
  threads before returning.

## Building

The project builds a single payload jar (`Relapse.jar`). You need:

- **JDK 8** (`javac` / `jar`) — the sources target `-source 1.4 -target 1.4`, which
  newer JDKs no longer support.
- A BD-J SDK providing `enhanced-stubs.zip`, `bdjstack.jar` and `rt.jar` under
  `target/lib/` (see [john-tornblom/bdj-sdk](https://github.com/john-tornblom/bdj-sdk)
  for the toolchain layout).

With the SDK laid out (`host/jdk8`, `target/lib/*`):

```sh
make BDJSDK_HOME=/path/to/bdj-sdk
```

Or compile directly with a JDK 8 and the three jars in one folder:

```sh
javac -Xlint:-options -source 1.4 -target 1.4 \
  -cp "enhanced-stubs.zip:bdjstack.jar:rt.jar" \
  src/org/bdj/api/*.java src/org/bdj/*.java src/org/bdj/external/*.java
jar cfm Relapse.jar manifest.txt -C src/ .
```

(On Windows, use `;` instead of `:` in the classpath.)

The embedded `kexp_2026_10_26.bin` and `elfldr-ps5-0.27.elf` under
`src/org/bdj/external/` are bundled into the jar automatically.

## Usage

Load `Relapse.jar` with BD-JB5's remote jar loader. Once it runs, the ELF loader
listens on **port 9021** — send your payloads there. Progress is reported over the
on-console UDP logger.

## Known issues

- Closing the application after running currently reboots the console during payload
  resource teardown; power-cycling / reconnecting is the expected workflow for now.

## Credits & acknowledgements

- **EchoStretch, zecoxao, Claude** — BD-J port.
- **relapse** — original PS5 kernel exploit (`aio_multi_wait` mode-0 UAF).
- **ufm42** — [kexp](https://github.com/ufm42/kexp).
- **ps5-payload-dev** — [elfldr](https://github.com/ps5-payload-dev/elfldr).
- **john-tornblom** — [bdj-sdk](https://github.com/john-tornblom/bdj-sdk).
- **Gezine** — BD-JB5 loader.
