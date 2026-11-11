# Architecture Guide: Building a Concurrent HTTP/HTTPS Chunked Downloader in Go

This document outlines the architectural blueprint, phases, and design considerations for implementing a robust, zero-external-dependency concurrent file downloader using the Go standard library.

---

## High-Level Architecture Overview

A concurrent chunked downloader works by dividing a remote resource into distinct byte ranges, requesting those ranges concurrently over HTTP, and writing the incoming byte streams directly into specific disk offsets of a single target file.

```
                           +------------------------+
                           |  Phase 1: Remote Probe  |
                           |  (HEAD / Range Check)  |
                           +-----------+------------+
                                       |
                                       v
                           +------------------------+
                           | Phase 2: Disk & State  |
                           |  Allocation (Truncate) |
                           +-----------+------------+
                                       |
                                       v
                           +------------------------+
                           | Phase 3: Partitioning  |
                           |   (Job Queue Planning) |
                           +-----------+------------+
                                       |
                   +-------------------+-------------------+
                   |                   |                   |
                   v                   v                   v
           +---------------+   +---------------+   +---------------+
           |   Worker 1    |   |   Worker 2    |   |   Worker N    |
           | Range: A -> B |   | Range: C -> D |   | Range: E -> F |
           +-------+-------+   +-------+-------+   +-------+-------+
                   |                   |                   |
                   +-------------------+-------------------+
                                       |
                                       v
                           +------------------------+
                           | Phase 4: Positional    |
                           | Writes (file.WriteAt)  |
                           +-----------+------------+
                                       |
                                       v
                           +------------------------+
                           | Phase 5: Verification  |
                           |   & Sync (File Flush)  |
                           +------------------------+
```

---

## Phase 1: Probing & Capability Handshake

Before allocating disk space or spinning up goroutines, the application must determine if the remote endpoint supports partial byte downloads and verify file boundaries.

### 1. The Probe Request
- Issue an initial request without a body (typically `HEAD`, or a fallback `GET` with `Range: bytes=0-0`).
- Follow HTTP redirects automatically to acquire the final target resource URL.

### 2. Header Inspection
- **`Accept-Ranges: bytes`**: Informs the client that the server understands byte offsets.
- **`Content-Length`**: Provides the exact size in bytes required for boundary calculation and file pre-allocation.
- **`ETag` and `Last-Modified`**: Unique resource identifiers used to guarantee the remote file has not been replaced or altered between requests or resume sessions.

### 3. Fallback Mechanism
- If `Accept-Ranges` is omitted, or if `Content-Length` is missing (e.g., streaming/`Transfer-Encoding: chunked`), multi-threading is impossible.
- The downloader must dynamically downgrade to a single-stream sequential `GET` pipeline.

---

## Phase 2: Storage Preparation & Disk Allocation

To allow multiple goroutines to write simultaneously without corrupting the file or needing continuous file locking, disk space must be prepared upfront.

### 1. The Direct-to-Target Pattern
Rather than downloading parts into separate temporary files (`part0.tmp`, `part1.tmp`) and stitching them together later—which causes massive disk I/O bottlenecks on large files—the downloader writes directly into the final destination file.

### 2. File Pre-Allocation
- Open the target file in read/write mode (`os.O_CREATE | os.O_RDWR`).
- Call `Truncate(contentLength)` immediately:
  - On modern filesystems (ext4, NTFS, APFS), this creates a sparse file or pre-allocates filesystem extents without writing zeroes across the whole disk.
  - Workers can instantly write at any arbitrary byte offset (e.g., offset 5,000,000) even before byte 0 is written.

### 3. State Initialization (Resumability)
- Check for a companion metadata file (e.g., `filename.meta.json`).
- If present, validate the stored `ETag` against the probe's `ETag`. If they match, load the list of uncompleted byte segments.
- If no state file exists, initialize a new record.

---

## Phase 3: Partitioning & Work Distribution

Chunking strategy directly affects recovery speed, network throughput, and memory consumption.

### 1. Chunk Sizing Strategy
Two primary distribution models exist:

| Strategy | Structure | Trade-off |
| :--- | :--- | :--- |
| **Fixed Split** | Divide total size by worker count (e.g., 4 workers = 4 huge segments). | Simplest to code, but if a connection stalls at 99%, the entire giant chunk must be restarted. |
| **Small-Block Queue** | Divide the file into smaller fixed blocks (e.g., 4 MiB to 16 MiB blocks). | More HTTP requests, but provides excellent resilience: failed requests only drop a few megabytes. |

### 2. Job Queue Setup
- Represent each piece of work as an immutable task unit:
  - `ChunkID`: Unique sequence index.
  - `StartOffset`: Beginning byte position (inclusive).
  - `EndOffset`: Terminal byte position (inclusive).
- Pass tasks into a buffered Go channel (`chan Chunk`).
- Launch a bounded pool of worker goroutines consuming from this single channel.

---

## Phase 4: Worker Execution & Positional I/O

Each worker goroutine processes one task at a time independently.

### 1. HTTP Range Request
- Construct a standard `GET` request.
- Attach the slice header: `Range: bytes={StartOffset}-{EndOffset}`.
- Inspect status code: The server **must** return `206 Partial Content`. A `200 OK` indicates the server ignored the range header and is streaming the whole file.

### 2. Positional Writes (`WriteAt`)
- Allocate a fixed-size byte buffer per worker (e.g., 32 KiB or 64 KiB) to minimize allocation churn.
- Stream data from `response.Body` into the buffer.
- Write each buffer slice using `os.File.WriteAt(buffer, currentOffset)`:
  - Unlike `os.File.Write()`, which modifies a shared seek pointer, `WriteAt` translates to POSIX `pwrite()`.
  - It is completely thread-safe across concurrent goroutines sharing the same `*os.File` pointer.

### 3. Progress Tracking & Retries
- Increment an atomic byte counter on each successful write to feed a progress bar or calculate download speed.
- Wrap network reads in a retry loop with exponential backoff:
  - If a socket times out or drops mid-chunk, update the chunk's start offset to `currentOffset` and re-issue the remaining range rather than throwing away the entire chunk.

---

## Phase 5: Verification, Flushing & Cleanup

Once the task queue is drained and workers terminate, the file must be formally closed and committed.

### 1. Integrity & Completeness Check
- Verify that the total number of bytes written matches the expected `Content-Length`.
- Ensure all chunks in the control queue completed without unresolved errors.

### 2. Filesystem Synchronization
- Call `file.Sync()` (`fsync` system call).
- This forces the operating system kernel to flush all dirty memory pages from the OS page cache to physical persistent storage (SSD/HDD).

### 3. Finalization
- Close the destination file handle.
- Delete the companion metadata state file.
- Present final summary metrics (total time, average bandwidth, total bytes transferred).

---

## Summary Checklist for Go Developers

- [ ] Use `net/http` with an explicit `http.Client{Timeout: ...}`.
- [ ] Check for `Accept-Ranges: bytes` and a valid positive `Content-Length`.
- [ ] Use `file.Truncate(size)` to pre-allocate storage.
- [ ] Use `file.WriteAt(buf, offset)` to prevent race conditions during concurrent writes.
- [ ] Never rely on `file.Seek()` or sequential `file.Write()` with multiple goroutines.
- [ ] Call `file.Sync()` before closing to prevent data loss on sudden power interruptions.