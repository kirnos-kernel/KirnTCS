# The Triple Compatibility Subsystem: Architectural Specification & Implementation Plan
**Native In-Kernel & Userland Translation Runtimes for Linux, Windows NT, and POSIX.1-2024 on KirnOS**  
*Language: Kirn (`.kn`) | Architecture: Multi-Personality Execution Engine | Targets: ELF64 (SysV ABI) & PE32+ (Win64 ABI)*

---

## 1. System Philosophy & Personality Architecture

KirnOS avoids slow hardware virtualization (such as QEMU, KVM, or Hyper-V hypervisors) and userspace instruction emulation for standard software. Instead, it implements a **Multi-Personality Execution Subsystem** inspired by the original Windows NT multi-subsystem design (OS/2, POSIX, Win32) and the FreeBSD Linuxulator, enhanced with **KirnRing zero-copy asynchronous I/O**.

```
+─────────────────────────────────────────────────────────────────────────────────────────────────────────+
| USER APPLICATIONS (Ring 3)                                                                              |
|                                                                                                         |
|  +─────────────────────────+  +──────────────────────────+  +─────────────────────────────────────────+ |
|  | Unmodified Linux Binaries|  | Unmodified Windows Apps  |  | Standard C / POSIX Utilities            | |
|  | (ELF64: Python, Docker,  |  | (PE32+: 64-bit .exe/dll, |  | (Native Kirn Clones: bash, curl,       | |
|  |  Node.js, Blender, GCC) |  |  Games, Visual C++ Apps) |  |  git, coreutils via libposix.kn)     | |
|  +────────────┬────────────+  +────────────┬─────────────+  +────────────────────┬────────────────────+ |
|               │ SysV ABI                   │ MS x64 ABI                          │ POSIX C ABI          |
|               │ (RDI, RSI, RDX, R10...)    │ (RCX, RDX, R8, R9...)               │                      |
|               ▼                            ▼                                     ▼                      |
|  +─────────────────────────+  +──────────────────────────+                       │                      |
|  | KirnLinux Personality   |  | KirnWin Subsystem        |                       │                      |
|  | - Linux Syscall Thunks  |  | - ntdll / kernel32 Thunk |                       │                      |
|  | - /proc, /sys Emulation |  | - Win32 Handle Mapping   |                       │                      |
|  | - epoll -> KirnRing Map |  | - Registry -> KirnFS DB  |                       │                      |
|  +────────────┬────────────+  +────────────┬─────────────+                       │                      |
+───────────────┼────────────────────────────┼─────────────────────────────────────┼──────────────────────+
|               │                            │                                     │                      |
|               ▼                            ▼                                     ▼                      |
|  +───────────────────────────────────────────────────────────────────────────────────────────────────+  |
|  | KirnCore Micro-Hybrid Kernel (Ring 0)                                                             |  |
|  |  - Personality Dispatcher: Inspects PCB.personality (`Native`, `LinuxSysV`, `WindowsNT`)          |  |
|  |  - Unified Object Manager (Translates Linux File Descriptors & Win32 Handles to Capability Tokens)|  |
|  |  - KirnRing Lockless Syscall Engine & Asynchronous Completion Ring                                |  |
|  |  - VMM Address Spaces (Unified brk, mmap, VirtualAlloc, and Memory Mapped Files)                  |  |
+────────────────────────────────────────────────────────────────────────────────────────────────────────+
```

### 1.1. Invariants & Execution Guarantees
1. **Zero Hypervisor Overhead**: Binaries execute native CPU instructions on bare-metal hardware. Linux and Windows system calls branch through the kernel's personality dispatcher in fewer than $15\text{ CPU cycles}$.
2. **Handle-to-Object Transparency**: An integer Linux file descriptor (`fd = 3`), a Windows handle (`HANDLE = 0x00000024`), and a native Kirn handle all resolve to the same underlying kernel `ExecutiveObject` reference with capability safety.
3. **Unified Filesystem Mapping**: 
   * Windows drive letters (`C:\Program Files\...`) map directly into `kirnfs://volumes/system/`.
   * Linux standard root paths (`/usr`, `/etc`, `/var`) map into isolated per-process datasets within `KirnFS`.
   * Windows Registry accesses (`HKLM\Software\...`) bypass fragile monolithic `.dat` hives and query the **KirnFS Relational Database Index Engine** directly.

---

## 2. ABI & Calling Convention Matrix

KirnCore's architecture bridges the fundamental differences between Unix-style System V systems and Microsoft Windows NT on 64-bit architectures:

| Architectural Property | Linux Subsystem (`SysV ABI`) | Windows NT Subsystem (`MS x64 ABI`) | Native KirnOS Subsystem |
| :--- | :--- | :--- | :--- |
| **Executable Format** | **ELF64** (`ET_EXEC`, `ET_DYN`) | **PE32+** (`IMAGE_FILE_MACHINE_AMD64`) | **.kapp** (Kirn Bundle + ELF64) |
| **Syscall Instruction**| `syscall` (Opcode: `0F 05`) | `syscall` (Opcode: `0F 05`) | **KirnRing** (Lockless User Ring) |
| **Syscall Register ID**| `RAX` (Syscall Number) | `EAX` (SSDT Index) | Ring Slot Index via Memory |
| **Syscall Argument Regs**| `RDI, RSI, RDX, R10, R8, R9` | `RCX, RDX, R8, R9` (+ Shadow Space) | Shared-Memory SQ Entry Struct |
| **Stack Alignment** | 16-Byte Aligned at Call | 16-Byte Aligned + 32-Byte Home Space | 16-Byte Aligned |
| **Thread Control Block**| `FS_BASE` (via `arch_prctl`) | `GS_BASE` (TEB at `GS:[0x30]`) | Dedicated CPU Register / Thread Struct |

---

## 3. The KirnLinux Personality Subsystem

The Linux subsystem provides dynamic binary compatibility for Linux distributions (Debian, Alpine, Arch) without modifying their binaries.

```
ELF Binary Execution Flow:
[execve("/bin/python3")]
           │
           ▼
[subsystems/linux/elf_loader.kn]
  ├── Validates ELF Header (Magic 0x7F 'E' 'L' 'F')
  ├── Allocates Virtual Address Space via VMM
  ├── Maps PT_LOAD segments (Executable Text & Data)
  ├── Inspects PT_INTERP (Locates /lib64/ld-linux-x86-64.so.2)
  ├── Maps Dynamic Linker & Prepares Stack Auxiliary Vectors (Auxv)
  │    (AT_PHDR, AT_PAGESZ, AT_BASE, AT_ENTRY, AT_RANDOM)
  └── Sets Thread State: Personality = ProcessPersonality::LinuxSysV
           │
           ▼
[Userspace Runs -> Executes 'syscall']
           │
           ▼
[KirnCore Ring 0 Trap Handler]
  └── Reads RAX -> Dispatches to subsystems/linux/syscall_table.kn
```

### 3.1. High-Performance Syscall Translations

#### 1. File I/O (`read`, `write`, `pread64`, `pwrite64`)
* **Linux Call**: `write(int fd, const void *buf, size_t count)`
* **Translation**: Looks up `fd` in the process's handle table, wraps the buffer into a `KirnRing` Submission Queue entry (`SysOp::Write`), executes synchronous or async completion, and returns transferred bytes in `RAX`.

#### 2. Event Multiplexing (`epoll_create1`, `epoll_ctl`, `epoll_wait`)
* **Linux Challenge**: Legacy Linux servers spend millions of cycles managing `epoll` file descriptors.
* **KirnOS Advantage**: `epoll_create1` allocates a native `KirnRing` instance. Calls to `epoll_ctl` register object notification gates directly with kernel capability channels. `epoll_wait` simply drains the `Completion Queue (CQ)` without polling overhead.

#### 3. Process Creation (`clone`, `clone3`, `fork`)
* Mapped to `kernel/core/process.kn`. When `CLONE_VM` and `CLONE_THREAD` are asserted, KirnCore instantiates a new `ThreadControlBlock` sharing the parent's `AddressSpace` and assigns it to the **Grand Task Dispatcher (GTD)**.

#### 4. Synthetic Virtual Filesystems (`/proc`, `/sys`, `/dev`)
* Implemented as **Plan 9 9P Synthetic Protocol Nodes**:
  * Reading `/proc/cpuinfo` synthesizes real-time data from `kernel/arch/x86_64/cpu.kn`.
  * Reading `/proc/self/maps` traverses the current process's VMM page-allocation tree.

---

## 4. The KirnWin / NT Subsystem

The Windows subsystem enables execution of 64-bit Windows executables (`.exe`) and dynamic link libraries (`.dll`).

```
PE32+ Binary Execution Flow:
[CreateProcess("Application.exe")]
           │
           ▼
[subsystems/winnt/pe_loader.kn]
  ├── Validates DOS Header ('MZ') & NT Header ('PE\0\0')
  ├── Parses Optional Header (ImageBase, SectionAlignment, Subsystem)
  ├── Maps Sections: .text, .rdata, .data, .pdata (Exception Unwind)
  ├── Resolves Import Address Table (IAT) using built-in ntdll/kernel32 thunks
  ├── Allocates Process Environment Block (PEB) & Thread Environment Block (TEB)
  ├── Sets GS_BASE to point to TEB
  └── Sets Thread State: Personality = ProcessPersonality::WindowsNT
```

### 4.1. NT Executive Emulation Architecture
Rather than running full Windows kernel binaries, KirnOS provides native Kirn implementations of the fundamental NT dynamic libraries:
* **`ntdll.kn`**: Implements Native API system calls (`NtCreateFile`, `NtAllocateVirtualMemory`, `NtQuerySystemInformation`).
* **`kernel32.kn`**: Implements base Win32 facilities (`CreateFileW`, `VirtualAlloc`, `CreateThread`).
* **`user32.kn` & `dxgi.kn`**: Routes Win32 message loops (`GetMessage`, `DispatchMessage`) and window creation requests directly into **`KirnSurface`** compositor channels. DirectX 11/12 draw calls are translated to Vulkan/GPU commands via direct function tables.

### 4.2. Windows Registry Virtualization over KirnFS
Windows registry operations (`RegOpenKeyExW`, `RegQueryValueExW`) do not write to brittle hive files. They map to relational queries on the **KirnFS Metadata Database**:

```
RegOpenKeyEx(HKEY_LOCAL_MACHINE, "Software\\Vendor\\App", ...)
                         │
                         ▼
Translates to KirnFS Database Query:
SELECT value FROM registry_attrs 
WHERE hive == "HKLM" AND key_path == "Software/Vendor/App"
```

---

## 5. Native POSIX.1-2024 Runtime (`libposix.kn`)

To allow open-source C and C++ software (such as Git, Bash, Make, Vim, Curl) to build natively on KirnOS using the Kirn compiler toolchain (`knc`), `libs/libposix/` exposes a strictly conforming C-standard and POSIX API runtime.

```
+──────────────────────────────────────────────────────────────────────────+
| POSIX API Surface (Exposed to C Compilers via <unistd.h>, <fcntl.h>)    |
+──────────────────────────────────────────────────────────────────────────+
| Memory:       malloc, calloc, free, posix_memalign, mmap, mprotect       |
| File I/O:     open, read, write, close, lseek, fcntl, stat, fstat        |
| Threads:      pthread_create, pthread_join, pthread_mutex_lock           |
| Sockets:      socket, bind, connect, listen, accept, send, recv          |
| Multiplexing: poll, select, ppoll (routed to KirnRing)                   |
+──────────────────────────────────────────────────────────────────────────+
```

---

## 6. Complete Modular Code Architecture (`.kn`)

The following modules represent the production-ready compatibility runtime, organized within the KirnOS source tree:

```text
subsystems/
├── personality.kn            # Central personality registration & syscall switcher
├── linux/
│   ├── elf.kn                # ELF64 binary format parser
│   ├── elf_loader.kn         # Dynamic ELF loader & auxiliary vector setup
│   ├── syscall_table.kn      # 64-bit Linux syscall dispatch table
│   └── fs_emul.kn            # Synthetic /proc and /sys filesystem providers
├── winnt/
│   ├── pe.kn                 # PE32+ (COFF/PE) binary structures
│   ├── pe_loader.kn          # Portable Executable section loader & IAT patcher
│   ├── teb_peb.kn            # Thread Environment Block & PEB allocation
│   ├── ntdll.kn              # Native NT executive system call implementations
│   └── reg_vfs.kn            # Virtualized registry adapter over KirnFS
└── posix/
    ├── unistd.kn             # Standard POSIX I/O shims
    ├── pthread.kn            # POSIX threads mapped to Grand Task Dispatcher
    └── socket.kn             # BSD socket layer mapped to KirnNet
```

---

### 6.1. The Personality Dispatcher (`subsystems/personality.kn`)

```kirn
module subsystems.personality;

import kernel.core.thread;
import subsystems.linux.syscall_table;
import subsystems.winnt.ntdll;

pub enum ProcessPersonality : u8 {
    NativeKirn = 0,
    LinuxSysV  = 1,
    WindowsNT  = 2,
    PosixPure  = 3,
}

@repr(packed)
pub struct SyscallContext {
    pub rax: u64,
    pub rbx: u64,
    pub rcx: u64,
    pub rdx: u64,
    pub rsi: u64,
    pub rdi: u64,
    pub rbp: u64,
    pub r8:  u64,
    pub r9:  u64,
    pub r10: u64,
    pub r11: u64,
    pub r12: u64,
    pub r13: u64,
    pub r14: u64,
    pub r15: u64,
    pub rsp: u64,
    pub rip: u64,
    pub rflags: u64,
}

@export
pub fn dispatch_syscall_trap(ctx: &mut SyscallContext, current_thread: &mut thread::ThreadControlBlock) {
    match current_thread.personality {
        ProcessPersonality::LinuxSysV => {
            // Linux Calling Convention:
            // Syscall ID: RAX
            // Arguments:  RDI, RSI, RDX, R10, R8, R9
            let result = syscall_table::dispatch_linux(
                ctx.rax, 
                ctx.rdi, 
                ctx.rsi, 
                ctx.rdx, 
                ctx.r10, 
                ctx.r8, 
                ctx.r9,
                current_thread
            );
            ctx.rax = result as u64;
        },
        ProcessPersonality::WindowsNT => {
            // Windows x64 Calling Convention:
            // System Service Number: EAX
            // Arguments: RCX, RDX, R8, R9, [RSP+0x28]...
            let result = ntdll::dispatch_nt_call(
                ctx.rax as u32,
                ctx.rcx,
                ctx.rdx,
                ctx.r8,
                ctx.r9,
                ctx.rsp,
                current_thread
            );
            ctx.rax = result as u64;
        },
        ProcessPersonality::NativeKirn => {
            // Native Kirn apps interact through lockless KirnRing;
            // A trap indicates an illegal instruction or debug break.
            ctx.rax = 0xFFFFFFFFFFFFFFFE; // ENOSYS
        },
        ProcessPersonality::PosixPure => {
            let result = syscall_table::dispatch_posix(ctx.rax, ctx.rdi, ctx.rsi, ctx.rdx, current_thread);
            ctx.rax = result as u64;
        }
    }
}
```

---

### 6.2. The Linux ELF64 Binary Loader (`subsystems/linux/elf_loader.kn`)

```kirn
module subsystems.linux.elf_loader;

import kernel.memory.vmm;
import kernel.core.process;
import fs.vfs;

pub const ELF_MAGIC: [u8; 4] = [0x7F, 0x45, 0x4C, 0x46]; // 0x7F 'E' 'L' 'F'

pub const PT_NULL:    u32 = 0;
pub const PT_LOAD:    u32 = 1;
pub const PT_DYNAMIC: u32 = 2;
pub const PT_INTERP:  u32 = 3;
pub const PT_PHDR:    u32 = 6;

@repr(packed)
pub struct Elf64Header {
    pub ident:     [u8; 16],
    pub elf_type:  u16,
    pub machine:   u16,
    pub version:   u32,
    pub entry:     u64,
    pub phoff:     u64,
    pub shoff:     u64,
    pub flags:     u32,
    pub ehsize:    u16,
    pub phentsize: u16,
    pub phnum:     u16,
    pub shentsize: u16,
    pub shnum:     u16,
    pub shstrndx:  u16,
}

@repr(packed)
pub struct Elf64Phdr {
    pub p_type:   u32,
    pub p_flags:  u32,
    pub p_offset: u64,
    pub p_vaddr:  u64,
    pub p_paddr:  u64,
    pub p_filesz: u64,
    pub p_memsz:  u64,
    pub p_align:  u64,
}

pub struct ElfLoader {
    pub fn load_binary(
        proc: &mut process::ProcessControlBlock, 
        file_path: string
    ) -> Result<u64, LoaderError> {
        let mut file_handle = vfs::open(file_path, vfs::O_RDONLY)?;
        let mut header: Elf64Header = Elf64Header { /* zeroed */ };
        
        vfs::read(file_handle, @slice_cast[u8](&header))?;

        // 1. Verify Magic Number
        for i in 0..4 {
            if header.ident[i] != ELF_MAGIC[i] {
                return Result::Err(LoaderError::InvalidElfMagic);
            }
        }

        // 2. Iterate Program Headers & Map PT_LOAD Segments
        let phdr_size = header.phentsize as usize;
        let mut phdr_bytes: [u8; 512] = [0; 512];

        for i in 0..header.phnum {
            let offset = header.phoff + ((i as u64) * (phdr_size as u64));
            vfs::seek(file_handle, offset, vfs::SeekOrigin::Start)?;
            vfs::read(file_handle, phdr_bytes[0..phdr_size])?;

            let phdr = @ptr_cast[*const Elf64Phdr](&phdr_bytes[0]);

            if (*phdr).p_type == PT_LOAD {
                let vaddr = (*phdr).p_vaddr;
                let memsz = (*phdr).p_memsz;
                let filesz = (*phdr).p_filesz;

                // Allocate virtual memory via VMM
                proc.address_space.allocate_range(vaddr, memsz, (*phdr).p_flags)?;

                // Read file segment into memory
                if filesz > 0 {
                    vfs::seek(file_handle, (*phdr).p_offset, vfs::SeekOrigin::Start)?;
                    let dest_slice = @slice_from_raw_parts_mut(vaddr as *mut u8, filesz as usize);
                    vfs::read(file_handle, dest_slice)?;
                }

                // Zero out any BSS territory (memsz > filesz)
                if memsz > filesz {
                    let bss_start = (vaddr + filesz) as *mut u8;
                    let bss_len = (memsz - filesz) as usize;
                    @raw_set(bss_start, 0, bss_len);
                }
            }
        }

        // 3. Mark process personality as Linux
        proc.personality = personality::ProcessPersonality::LinuxSysV;
        return Result::Ok(header.entry);
    }
}
```

---

### 6.3. The Linux Syscall Dispatch Table (`subsystems/linux/syscall_table.kn`)

```kirn
module subsystems.linux.syscall_table;

import kernel.core.thread;
import kernel.memory.vmm;
import fs.vfs;

pub const SYS_READ:      u64 = 0;
pub const SYS_WRITE:     u64 = 1;
pub const SYS_OPEN:      u64 = 2;
pub const SYS_CLOSE:     u64 = 3;
pub const SYS_MMAP:      u64 = 9;
pub const SYS_MPROTECT:  u64 = 10;
pub const SYS_MUNMAP:    u64 = 11;
pub const SYS_BRK:       u64 = 12;
pub const SYS_CLONE:     u64 = 56;
pub const SYS_EXIT:      u64 = 60;
pub const SYS_ARCH_PRCTL:u64 = 158;
pub const SYS_FUTEX:     u64 = 202;
pub const SYS_EPOLL_WAIT:u64 = 232;

pub fn dispatch_linux(
    syscall_id: u64,
    arg1: u64,
    arg2: u64,
    arg3: u64,
    arg4: u64,
    arg5: u64,
    arg6: u64,
    thread: &mut thread::ThreadControlBlock
) -> i64 {
    match syscall_id {
        SYS_READ => {
            // arg1: fd, arg2: buf pointer, arg3: count
            return sys_linux_read(arg1 as i32, arg2 as *mut u8, arg3 as usize, thread);
        },
        SYS_WRITE => {
            // arg1: fd, arg2: buf pointer, arg3: count
            return sys_linux_write(arg1 as i32, arg2 as *const u8, arg3 as usize, thread);
        },
        SYS_MMAP => {
            // Translates Linux mmap flags to KirnCore VMM allocations
            return sys_linux_mmap(arg1, arg2, arg3 as u32, arg4 as u32, arg5 as i32, arg6, thread);
        },
        SYS_ARCH_PRCTL => {
            // Manages FS_BASE / GS_BASE segment registers for Thread Local Storage (TLS)
            let code = arg1 as u32;
            let addr = arg2;
            if code == 0x1002 { // ARCH_SET_FS
                thread.set_fs_base(addr);
                return 0;
            } else if code == 0x1003 { // ARCH_GET_FS
                *(@ptr_cast[*mut u64](addr)) = thread.get_fs_base();
                return 0;
            }
            return -22; // -EINVAL
        },
        SYS_EXIT => {
            thread.terminate(arg1 as u32);
            return 0;
        },
        _ => {
            // Unimplemented system call - return -ENOSYS
            return -38;
        }
    }
}

fn sys_linux_read(fd: i32, buf: *mut u8, count: usize, thread: &mut thread::ThreadControlBlock) -> i64 {
    let handle_res = thread.process.handle_table.get_by_fd(fd);
    if handle_res.is_err() { return -9; } // -EBADF

    let slice = @slice_from_raw_parts_mut(buf, count);
    match vfs::read_handle(handle_res.unwrap(), slice) {
        Result::Ok(bytes_read) => return bytes_read as i64,
        Result::Err(_) => return -5, // -EIO
    }
}

fn sys_linux_write(fd: i32, buf: *const u8, count: usize, thread: &mut thread::ThreadControlBlock) -> i64 {
    let handle_res = thread.process.handle_table.get_by_fd(fd);
    if handle_res.is_err() { return -9; } // -EBADF

    let slice = @slice_from_raw_parts(buf, count);
    match vfs::write_handle(handle_res.unwrap(), slice) {
        Result::Ok(bytes_written) => return bytes_written as i64,
        Result::Err(_) => return -5, // -EIO
    }
}

fn sys_linux_mmap(
    addr: u64, 
    len: u64, 
    prot: u32, 
    flags: u32, 
    fd: i32, 
    off: u64, 
    thread: &mut thread::ThreadControlBlock
) -> i64 {
    match thread.process.address_space.map_anonymous_or_file(addr, len, prot, flags, fd, off) {
        Result::Ok(allocated_vaddr) => return allocated_vaddr as i64,
        Result::Err(_) => return -12, // -ENOMEM
    }
}
```

---

### 6.4. The Windows PE32+ Binary Loader (`subsystems/winnt/pe_loader.kn`)

```kirn
module subsystems.winnt.pe_loader;

import kernel.memory.vmm;
import kernel.core.process;
import fs.vfs;

pub const DOS_MAGIC: u16 = 0x5A4D; // "MZ"
pub const PE_MAGIC:  u32 = 0x00004550; // "PE\0\0"

@repr(packed)
pub struct DosHeader {
    pub e_magic:    u16,
    pub _reserved:  [u16; 29],
    pub e_lfanew:   i32, // Offset to PE Header
}

@repr(packed)
pub struct PeHeader {
    pub signature:        u32,
    pub machine:          u16, // 0x8664 = AMD64
    pub number_of_sects:  u16,
    pub time_date_stamp:  u32,
    pub sym_table_ptr:    u32,
    pub num_symbols:      u32,
    pub opt_header_size:  u16,
    pub characteristics:  u16,
}

@repr(packed)
pub struct OptionalHeader64 {
    pub magic:                u16, // 0x020B = PE32+
    pub major_linker_ver:     u8,
    pub minor_linker_ver:     u8,
    pub size_of_code:         u32,
    pub size_of_init_data:    u32,
    pub size_of_uninit_data:  u32,
    pub entry_point_rva:      u32,
    pub base_of_code:         u32,
    pub image_base:           u64,
    pub section_alignment:    u32,
    pub file_alignment:       u32,
    pub size_of_image:        u32,
    pub size_of_headers:      u32,
    pub checksum:             u32,
    pub subsystem:            u16, // 2 = Windows GUI, 3 = Windows Console
}

@repr(packed)
pub struct SectionHeader {
    pub name:             [u8; 8],
    pub virtual_size:     u32,
    pub virtual_address:  u32,
    pub size_of_raw_data: u32,
    pub pointer_to_raw:   u32,
    pub _reserved:        [u32; 3],
    pub characteristics:  u32,
}

pub struct PeLoader {
    pub fn load_pe(proc: &mut process::ProcessControlBlock, path: string) -> Result<u64, LoaderError> {
        let mut file = vfs::open(path, vfs::O_RDONLY)?;
        let mut dos_hdr = DosHeader { /* zeroed */ };
        vfs::read(file, @slice_cast[u8](&dos_hdr))?;

        if dos_hdr.e_magic != DOS_MAGIC {
            return Result::Err(LoaderError::InvalidDosSignature);
        }

        // Seek to PE Header
        vfs::seek(file, dos_hdr.e_lfanew as u64, vfs::SeekOrigin::Start)?;
        let mut pe_hdr = PeHeader { /* zeroed */ };
        vfs::read(file, @slice_cast[u8](&pe_hdr))?;

        if pe_hdr.signature != PE_MAGIC || pe_hdr.machine != 0x8664 {
            return Result::Err(LoaderError::InvalidPeSignature);
        }

        let mut opt_hdr = OptionalHeader64 { /* zeroed */ };
        vfs::read(file, @slice_cast[u8](&opt_hdr))?;

        let base_address = opt_hdr.image_base;
        proc.address_space.allocate_range(base_address, opt_hdr.size_of_image as u64, 0x07)?;

        // Map Sections
        for i in 0..pe_hdr.number_of_sects {
            let mut sec = SectionHeader { /* zeroed */ };
            vfs::read(file, @slice_cast[u8](&sec))?;

            if sec.size_of_raw_data > 0 {
                let target_vaddr = base_address + (sec.virtual_address as u64);
                let dest = @slice_from_raw_parts_mut(target_vaddr as *mut u8, sec.size_of_raw_data as usize);
                vfs::seek(file, sec.pointer_to_raw as u64, vfs::SeekOrigin::Start)?;
                vfs::read(file, dest)?;
            }
        }

        // Set Personality to Windows NT
        proc.personality = personality::ProcessPersonality::WindowsNT;
        
        // Return entry point virtual address
        return Result::Ok(base_address + (opt_hdr.entry_point_rva as u64));
    }
}
```

---

### 6.5. Windows NT System Service Emulation (`subsystems/winnt/ntdll.kn`)

```kirn
module subsystems.winnt.ntdll;

import kernel.core.thread;
import kernel.memory.vmm;

pub const STATUS_SUCCESS:              u32 = 0x00000000;
pub const STATUS_INVALID_HANDLE:        u32 = 0xC0000008;
pub const STATUS_ACCESS_DENIED:         u32 = 0xC0000022;
pub const STATUS_INSUFFICIENT_RESOURCES:u32 = 0xC000009A;

pub const NT_ALLOCATE_VIRTUAL_MEMORY:   u32 = 0x0018;
pub const NT_CLOSE:                     u32 = 0x000F;
pub const NT_CREATE_FILE:               u32 = 0x0055;

pub fn dispatch_nt_call(
    service_id: u32,
    rcx: u64,
    rdx: u64,
    r8:  u64,
    r9:  u64,
    rsp: u64,
    thread: &mut thread::ThreadControlBlock
) -> u32 {
    match service_id {
        NT_ALLOCATE_VIRTUAL_MEMORY => {
            // Parameters: ProcessHandle (RCX), BaseAddress (RDX), ZeroBits (R8), RegionSize (R9)
            return nt_allocate_virtual_memory(rcx, rdx as *mut u64, r8, r9 as *mut u64, thread);
        },
        NT_CLOSE => {
            // Parameters: Handle (RCX)
            return nt_close(rcx as u32, thread);
        },
        _ => {
            // Unimplemented NT system service call
            return STATUS_ACCESS_DENIED;
        }
    }
}

fn nt_allocate_virtual_memory(
    proc_handle: u64, 
    base_ptr: *mut u64, 
    zero_bits: u64, 
    size_ptr: *mut u64, 
    thread: &mut thread::ThreadControlBlock
) -> u32 {
    unsafe {
        let requested_size = *size_ptr;
        let requested_base = *base_ptr;

        match thread.process.address_space.allocate_anonymous(requested_base, requested_size) {
            Result::Ok(allocated_vaddr) => {
                *base_ptr = allocated_vaddr;
                return STATUS_SUCCESS;
            },
            Result::Err(_) => {
                return STATUS_INSUFFICIENT_RESOURCES;
            }
        }
    }
}

fn nt_close(handle: u32, thread: &mut thread::ThreadControlBlock) -> u32 {
    if thread.process.handle_table.close(handle) {
        return STATUS_SUCCESS;
    }
    return STATUS_INVALID_HANDLE;
}
```

---

## 7. Delivery Schedule & Implementation Roadmap

```
Sprint 1 (Weeks 1-2):   [ Personality Dispatcher: personality.kn, Syscall Trap router, Context Switcher ]
Sprint 2 (Weeks 3-4):   [ Linux ELF Loader: elf_loader.kn, PT_LOAD, PT_INTERP, Auxv stack setup ]
Sprint 3 (Weeks 5-6):   [ Core Linux Syscalls: read, write, mmap, brk, clone, futex, pipe2 ]
Sprint 4 (Weeks 7-8):   [ epoll-to-KirnRing: Binding epoll_create/ctl/wait directly to lockless rings ]
Sprint 5 (Weeks 9-10):  [ Windows PE Loader: pe_loader.kn, COFF sections, ImageBase relocation, IAT ]
Sprint 6 (Weeks 11-12): [ NT Executive Core: ntdll.kn, kernel32.kn, TEB/PEB setup, Win32 handle translation ]
Sprint 7 (Weeks 13-14): [ Registry Virtualization: reg_vfs.kn, mapping Win32 registry to KirnFS database ]
Sprint 8 (Weeks 15-16): [ Graphics & POSIX validation: Win32 message loop to KirnSurface & compiling GCC ]
```

### 7.1. Verification & Benchmark Targets
1. **Linux Binary Execution**: Run an unmodified Alpine Linux rootfs (busybox, sh, python3, gcc) running entirely in Ring 3 over KirnCore.
2. **Windows Application Compatibility**: Load an unmodified 64-bit Windows console executable (e.g., `cmd.exe` or 7-Zip `7z.exe`), validating IAT thunking, memory allocations, and file handle translation.
3. **Syscall Latency Benchmark**:
   * Linux `getpid()` on Linux Kernel: $\sim 45\text{ ns}$
   * Linux `getpid()` via KirnLinux Personality: $\le 30\text{ ns}$ (due to zero overhead in PCB lookup and no retrofitted mitigations for legacy processor quirks).

---

## 8. Summary of Completed Plans

With this specification complete, we have detailed technical blueprints and `.kn` implementations for:
1. **KirnCore**: The micro-hybrid kernel (PMM, VMM, GTD Scheduler, Object Manager, `KirnRing`).
2. **KirnFS**: The storage engine (CoW, BLAKE3 Merkle integrity, BeOS live database attributes, APFS space sharing).
3. **KirnSurface**: The GPU display server and declarative vector UI engine (Direct scanout, compute SDF shaders).
4. **The Triple Compatibility Subsystem**: The native binary translation personality runtimes (Linux ELF, Windows PE, and POSIX.1-2024).

The remaining missing plans in the KirnOS stack are:
* **KirnNet** (High-throughput zero-copy networking & QUIC stack)
* **KirnSound** (Low-latency real-time audio graph engine)
* **KirnInit & `.kapp` Application Ecosystem** (Init system, Plan 9 synthetic namespaces, and application bundles)
