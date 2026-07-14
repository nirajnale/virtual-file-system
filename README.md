# Virtual File System

A UNIX-like virtual file system implemented in C++ from scratch.
Built to understand how operating systems manage files internally — inode structures,
file descriptor tables, memory-backed storage, and system calls — without relying on
any external library or OS abstraction.

---

## Overview

Most developers use file systems without understanding what happens underneath.
This project implements one.

The system runs as an interactive shell where users create, open, read, write, seek,
and delete files entirely in memory. Every concept — inodes, file descriptors,
permissions, reference counts, link counts — is implemented explicitly rather than
borrowed from the OS.

```
Customised VFS : > create notes.txt 3
Customised VFS : > open notes.txt 2
Customised VFS : > write notes.txt
Enter data : Backend engineering is about understanding systems deeply.
Customised VFS : > read notes.txt 100
Backend engineering is about understanding systems deeply.
Customised VFS : > stat notes.txt
```

---

## Architecture

The system is built from four cooperating data structures:

```
┌─────────────────────────────────────────────────────────┐
│                    SUPERBLOCK                           │
│         TotalInodes: 50  │  FreeInodes: (tracked)      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              DILB  (Doubly Indirect List Block)         │
│         Linked list of all inode structures             │
│  [inode 1] → [inode 2] → [inode 3] → ... → [inode 50] │
└─────────────────────────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
┌─────────────────────┐   ┌───────────────────────────┐
│       INODE         │   │       FILE TABLE           │
│  FileName           │◄──│  readoffset               │
│  InodeNumber        │   │  writeoffset              │
│  FileSize           │   │  mode (read/write/r+w)    │
│  FileActualSize     │   │  count (open references)  │
│  FileType           │   │  ptrinode ──────────────► │
│  LinkCount          │   └───────────────────────────┘
│  ReferenceCount     │                 ▲
│  permission (1/2/3) │                 │
│  *Buffer (data)     │   ┌─────────────────────────────┐
│  *next              │   │    UFDT  (User File          │
└─────────────────────┘   │    Descriptor Table)         │
                          │  UFDTArr[50]                 │
                          │  index = file descriptor     │
                          │  ptrfiletable ──────────────►│
                          └─────────────────────────────┘
```

**Superblock** tracks total and free inodes across the system.

**DILB (Doubly Indirect List Block)** is a linked list of all 50 inode structures,
allocated at startup. Each inode holds file metadata and a pointer to the
file's in-memory data buffer.

**File Table** is created per `open()` call. Tracks read/write offsets, open mode,
and a reference count. Multiple opens of the same file create separate file table
entries pointing to the same inode.

**UFDT (User File Descriptor Table)** is a fixed array of 50 slots. Each slot holds
a pointer to a file table entry. The array index is the file descriptor returned
to the caller.

---

## System Calls

| Command | Signature | Description |
|---|---|---|
| `create` | `create <name> <permission>` | Allocates a free inode, sets file name and permission |
| `open` | `open <name> <mode>` | Creates a file table entry, assigns a file descriptor |
| `close` | `close <name>` | Decrements reference count, frees file table if zero |
| `closeall` | `closeall` | Closes all open file descriptors |
| `read` | `read <name> <bytes>` | Reads N bytes from current read offset |
| `write` | `write <name>` | Writes stdin input to current write offset |
| `lseek` | `lseek <name> <offset> <origin>` | Repositions read/write offset |
| `stat` | `stat <name>` | Displays inode metadata by file name |
| `fstat` | `fstat <fd>` | Displays inode metadata by file descriptor |
| `truncate` | `truncate <name>` | Clears file data buffer, resets size to zero |
| `rm` | `rm <name>` | Decrements link count, frees inode when count reaches zero |
| `ls` | `ls` | Lists all files with type, size, link count, reference count |
| `man` | `man <command>` | Displays usage documentation for any command |
| `help` | `help` | Lists all available commands |

**Permission values:** `1` = read only · `2` = write only · `3` = read + write

**Open modes:** `1` = read · `2` = write · `3` = read + write

---

## Key Engineering Decisions

**Why a linked list for inodes (DILB)?**
Inodes are allocated at startup as a linked list rather than an array. This mirrors
how real file systems use a free list for inode management and allows traversal
to find the first free inode by checking `FileType == 0`.

**Why separate file table and inode?**
The separation models the UNIX design precisely: multiple processes (or multiple
`open()` calls) can hold independent file descriptors with independent offsets
pointing to the same underlying inode. Reference counting on the file table and
link counting on the inode handle cleanup correctly — the inode is only freed
when all links and all references reach zero.

**Why in-memory storage?**
The focus of this project is the structural design of a file system — descriptor
tables, inode management, permission enforcement, offset tracking — rather than
block-device I/O. Using a heap-allocated buffer per inode (`char *Buffer`) isolates
the interesting problem from disk I/O complexity.

**Why a custom shell command parser?**
The `main()` loop uses `sscanf` to tokenize input into up to four command tokens,
then dispatches to the appropriate system call. This models how a real kernel
receives and parses system call arguments.

---

## Project Structure

```
virtual-file-system/
│
├── SourceCode/
│   └── VirtualFileSystem.cpp   # Complete implementation (~850 lines)
│
└── README.md
```

All data structures, system call implementations, and the interactive shell
are contained in a single translation unit, reflecting the monolithic design
of a simplified kernel module.

---

## Build and Run

**Requirements:** GCC or G++ (C++11 or later), any Unix/Linux/Windows terminal

```bash
# Clone the repository
git clone https://github.com/nirajnale/virtual-file-system.git
cd virtual-file-system/SourceCode

# Compile
g++ VirtualFileSystem.cpp -o vfs

# Run
./vfs
```

**Windows:**
```bash
g++ VirtualFileSystem.cpp -o vfs.exe
vfs.exe
```

---

## Example Session

```
---DILB created successfully---

Customised VFS : > help
ls       : To List out all files
create   : To create a new file
open     : To open the file
close    : To close the file
read     : To Read the content from file
write    : To Write content into file
lseek    : To change the offset of the file
stat     : To Display information of file using name
fstat    : To Display information of file using file descriptor
truncate : To Remove all the data from file
rm       : To Delete the file

Customised VFS : > create report.txt 3
File created successfully

Customised VFS : > open report.txt 2
File descriptor : 0

Customised VFS : > write report.txt
Enter data : Q3 results: revenue up 18%, costs down 12%.

Customised VFS : > open report.txt 1
File descriptor : 1

Customised VFS : > read report.txt 50
Q3 results: revenue up 18%, costs down 12%.

Customised VFS : > stat report.txt
File name       : report.txt
Inode number    : 1
File size       : 1024
Actual size     : 42
Link count      : 1
Reference count : 2
Permission      : 3

Customised VFS : > ls
File Name     File Type    File Size    Link Count    Ref Count
report.txt    Regular      1024         1             2

Customised VFS : > lseek report.txt 0 1
Offset changed successfully

Customised VFS : > rm report.txt
File deleted successfully
```

---

## What This Demonstrates

- **inode-based file metadata management** — each file's metadata lives separately from its data, mirroring UNIX design
- **File descriptor indirection** — UFDT → file table → inode, the same three-layer model used in real kernels
- **Reference and link counting** — correct cleanup semantics when multiple descriptors point to the same file
- **Permission enforcement** — open mode validated against inode permission before reads and writes are permitted
- **Offset management** — independent read and write offsets per file table entry, repositionable via lseek
- **Free inode tracking** — superblock and DILB traversal to allocate and reclaim inodes

---

## License

Niraj Nale
