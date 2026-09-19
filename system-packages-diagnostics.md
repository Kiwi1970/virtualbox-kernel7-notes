# System package installation & diagnostics

**Host:** Linux Mint (Ubuntu 24.04 / noble base)  
**Kernel at check:** `7.0.0-31-generic`  
**Documented:** 2026-09-20T11:08:39+12:00

## Requested packages

An external/tooling check asked for these libraries:

- `libapr1`
- `libaprutil1`
- `libasound2`
- `libglib2.0-0`
- `libxcb-cursor` (also required earlier as `libxcb-cursor0`)

## Name mapping on Ubuntu 24.04 (noble)

On noble, several packages were renamed for the 64-bit `time_t` transition (`t64`). The old names have **no install candidate**.

| Requested name | Installable package |
|----------------|---------------------|
| `libapr1` | `libapr1t64` |
| `libaprutil1` | `libaprutil1t64` |
| `libasound2` | `libasound2t64` |
| `libglib2.0-0` | `libglib2.0-0t64` |
| `libxcb-cursor` | `libxcb-cursor0` |

## Install commands used

```bash
sudo apt-get update
sudo apt-get install -y \
  libapr1t64 \
  libaprutil1t64 \
  libasound2t64 \
  libglib2.0-0t64 \
  libxcb-cursor0

# Later reinstall to refresh packages:
sudo apt-get install -y --reinstall \
  libapr1t64 \
  libaprutil1t64 \
  libasound2t64 \
  libglib2.0-0t64
```

## Installed versions (verified)

| Package | Architecture | Version | dpkg status |
|---------|--------------|---------|-------------|
| `libapr1t64` | amd64 | `1.7.2-3.1ubuntu0.1` | `ii` (installed OK) |
| `libaprutil1t64` | amd64 | `1.6.3-1.1ubuntu7.1` | `ii` |
| `libasound2t64` | amd64 | `1.2.11-1ubuntu0.3` | `ii` |
| `libglib2.0-0t64` | amd64 | `2.80.0-6ubuntu3.8` | `ii` |
| `libxcb-cursor0` | amd64 | `0.1.4-1build1` | `ii` |

All installed versions matched their APT candidates (**up to date**).

## Diagnostic checks run

### 1. Package query

```bash
dpkg-query -W -f='${db:Status-Abbrev}\t${Package}\t${Architecture}\t${Version}\n' \
  libapr1t64 libaprutil1t64 libasound2t64 libglib2.0-0t64 libxcb-cursor0
```

**Result:** all `ii`.

### 2. dpkg audit

```bash
sudo dpkg --audit
```

**Result:** exit code `0` (no broken packages reported).

### 3. APT dependency check

```bash
sudo apt-get check
```

**Result:** exit code `0` (dependency tree OK).

### 4. Shared library resolution

`ldconfig` resolved:

| Library soname | Path |
|----------------|------|
| `libapr-1.so.0` | `/lib/x86_64-linux-gnu/libapr-1.so.0` |
| `libaprutil-1.so.0` | `/lib/x86_64-linux-gnu/libaprutil-1.so.0` |
| `libasound.so.2` | `/lib/x86_64-linux-gnu/libasound.so.2` |
| `libglib-2.0.so.0` | `/lib/x86_64-linux-gnu/libglib-2.0.so.0` |
| `libxcb-cursor.so.0` | `/lib/x86_64-linux-gnu/libxcb-cursor.so.0` |

Files are also present under `/usr/lib/x86_64-linux-gnu/`.

### 5. Package file lists

`dpkg -L` reported packaged files for each package (counts at time of check):

- `libapr1t64`: 15 files
- `libaprutil1t64`: 22 files
- `libasound2t64`: 16 files
- `libglib2.0-0t64`: 33 files
- `libxcb-cursor0`: 11 files

## Conclusion

**PASS.** The requested libraries are installed via their noble `t64` / current package names, APT/dpkg report a healthy system, and the shared objects are loadable.

## Re-run diagnostics

```bash
dpkg-query -W -f='${db:Status-Abbrev}\t${Package}\t${Version}\n' \
  libapr1t64 libaprutil1t64 libasound2t64 libglib2.0-0t64 libxcb-cursor0
sudo dpkg --audit
sudo apt-get check
ldconfig -p | grep -E 'libapr-1|libaprutil-1|libasound|libglib-2.0|libxcb-cursor'
```
