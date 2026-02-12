# mgrep -- Developer Guide

A concurrent grep clone built with Go goroutines and channels. This document walks through the **entire codebase** from program start to exit, explaining every function, every goroutine, every synchronization primitive, and why each design decision was made.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Data Types -- The Building Blocks](#3-data-types----the-building-blocks)
4. [Execution Flow -- Start to Finish](#4-execution-flow----start-to-finish)
   - [Phase 1: Argument Parsing](#phase-1-argument-parsing)
   - [Phase 2: Initializing the Worklist and Results Channel](#phase-2-initializing-the-worklist-and-results-channel)
   - [Phase 3: Directory Discovery Goroutine (Producer)](#phase-3-directory-discovery-goroutine-producer)
   - [Phase 4: Worker Goroutines (Consumers)](#phase-4-worker-goroutines-consumers)
   - [Phase 5: The Display Goroutine (Printer)](#phase-5-the-display-goroutine-printer)
   - [Phase 6: Waiting for Completion](#phase-6-waiting-for-completion)
5. [The Worklist Package -- Thread-Safe Job Queue](#5-the-worklist-package----thread-safe-job-queue)
6. [The Worker Package -- File Searching Logic](#6-the-worker-package----file-searching-logic)
7. [Why Goroutines? What Would Happen Without Them?](#7-why-goroutines-what-would-happen-without-them)
8. [Synchronization Deep Dive](#8-synchronization-deep-dive)
9. [Debugging Guide](#9-debugging-guide)
10. [How to Run](#10-how-to-run)

---

## 1. Project Structure

```
grep-clone-with-coroutines/
├── go.mod              # Module definition: "mgrep", Go 1.22.4
├── go.sum              # Dependency checksums
├── REQUIREMENTS        # Original project requirements
├── DEVELOPER_GUIDE.md  # This file
├── mgrep/
│   └── main.go         # Entry point -- orchestrates everything
├── worklist/
│   └── worklist.go     # Thread-safe job queue (channel-backed)
└── worker/
    └── worker.go       # File search logic (substring matching)
```

**Three packages:**
- `main` (in `mgrep/`) -- program entry, CLI parsing, goroutine orchestration
- `worklist` -- a channel-backed concurrent job queue
- `worker` -- pure file I/O and string matching, no concurrency awareness

---

## 2. High-Level Architecture

The program follows a **Producer-Consumer** pattern with a **display pipeline**:

```
┌─────────────────┐       ┌────────────────────┐       ┌───────────────┐
│   Discovery     │       │   Worker Pool       │       │   Display     │
│   Goroutine     │──────>│   (10 goroutines)   │──────>│   Goroutine   │
│   (Producer)    │ jobs  │   (Consumers)       │results│   (Printer)   │
└─────────────────┘ chan  └────────────────────┘  chan  └───────────────┘
                                                              │
                                                              ▼
                                                          stdout
```

- **1 Discovery goroutine** recursively walks directories and pushes file paths into a job channel.
- **10 Worker goroutines** pull file paths from the job channel, search each file, and push matches into a results channel.
- **1 Display goroutine** reads from the results channel and prints to the terminal.
- **The main goroutine** waits for everything to finish, then exits.

---

## 3. Data Types -- The Building Blocks

### `worklist.Entry` (`worklist/worklist.go`, line 3-5)

```go
type Entry struct {
    Path string
}
```

A single unit of work. `Path` holds the absolute or relative path to a file that needs to be searched. An **empty** `Path` (`""`) is a special **poison pill** -- it signals a worker to stop (more on this later).

### `worklist.Worklist` (`worklist/worklist.go`, line 7-9)

```go
type Worklist struct {
    jobs chan Entry
}
```

A thread-safe job queue. The `jobs` field is a **buffered channel** -- Go channels are inherently safe for concurrent reads and writes, so no mutexes are needed.

### `worker.Result` (`worker/worker.go`, line 10-14)

```go
type Result struct {
    Line    string
    LineNum int
    Path    string
}
```

A single grep match. Contains the matching line's text, its line number, and which file it came from.

### `worker.Results` (`worker/worker.go`, line 16-18)

```go
type Results struct {
    Inner []Result
}
```

A collection of all matches found in a single file. `Inner` is a slice of `Result`.

---

## 4. Execution Flow -- Start to Finish

### Phase 1: Argument Parsing

**File:** `mgrep/main.go`, lines 30-36

```go
var args struct {
    SearchTerm string `arg:"positional,required"`
    SearchDir  string `arg:"positional"`
}

func main() {
    arg.MustParse(&args)
```

The very first thing `main()` does is parse command-line arguments using the `go-arg` library. The struct tags define:

- `SearchTerm` -- **required**, first positional arg (the string to search for)
- `SearchDir` -- **optional**, second positional arg (the directory to search in; defaults to empty string which means current directory behavior depends on OS)

`arg.MustParse` will print usage info and `os.Exit(1)` if the required argument is missing. After this line, `args.SearchTerm` and `args.SearchDir` are populated.

**Usage:** `mgrep "func" "./src"`

---

### Phase 2: Initializing the Worklist and Results Channel

**File:** `mgrep/main.go`, lines 38-42

```go
var workersWg sync.WaitGroup

wl := worklist.New(100)
results := make(chan worker.Result, 100)
numWorkers := 10
```

Three things are created here:

| Variable      | Type               | Purpose |
|---------------|-------------------|---------|
| `workersWg`   | `sync.WaitGroup`   | Tracks when the discovery goroutine + all 10 worker goroutines have finished |
| `wl`          | `worklist.Worklist` | Buffered channel of 100 `Entry` items -- the job queue |
| `results`     | `chan worker.Result` | Buffered channel of 100 `Result` items -- the output pipeline |
| `numWorkers`  | `int`              | 10 concurrent file-searching goroutines |

**Why buffer size 100?** The buffer allows the producer (discovery) to push up to 100 jobs without blocking, even if consumers (workers) haven't started pulling yet. Similarly, workers can push up to 100 results without blocking, even if the display goroutine is busy printing. This decouples the speeds of each stage.

---

### Phase 3: Directory Discovery Goroutine (Producer)

**File:** `mgrep/main.go`, lines 44-50

```go
workersWg.Add(1)

go func() {
    defer workersWg.Done()
    discoverDirs(&wl, args.SearchDir)
    wl.Finalize(numWorkers)
}()
```

This launches the **first goroutine**. Let's break it down:

1. **`workersWg.Add(1)`** -- tells the WaitGroup: "one more goroutine is in flight."
2. **`go func() { ... }()`** -- spawns an anonymous goroutine.
3. **`defer workersWg.Done()`** -- when this goroutine exits (for any reason), decrement the WaitGroup counter by 1.
4. **`discoverDirs(&wl, args.SearchDir)`** -- recursively walk the directory tree and push every file as a job into the worklist.
5. **`wl.Finalize(numWorkers)`** -- after all files are discovered, send 10 "poison pill" entries (empty paths) to tell each worker to shut down.

#### The `discoverDirs` Function

**File:** `mgrep/main.go`, lines 14-28

```go
func discoverDirs(wl *worklist.Worklist, path string) {
    entries, err := os.ReadDir(path)
    if err != nil {
        fmt.Println("Error reading directory:", path)
        return
    }
    for _, entry := range entries {
        if entry.IsDir() {
            nextPath := filepath.Join(path, entry.Name())
            discoverDirs(wl, nextPath)
        } else {
            wl.Add(worklist.NewJob(filepath.Join(path, entry.Name())))
        }
    }
}
```

**Step-by-step:**

1. Read all entries in the given `path` using `os.ReadDir`.
2. If it fails (e.g., permission denied), print an error and return gracefully -- no crash.
3. For each entry:
   - **If it's a directory** -- recursively call `discoverDirs` to go deeper.
   - **If it's a file** -- create a new `Entry` with the file's full path and push it into the worklist channel via `wl.Add(...)`.

This is a **depth-first** recursive traversal. Files are pushed into the channel as they are found, meaning workers can start processing files **before** discovery is even complete. This is the power of the concurrent pipeline.

#### The Finalize Mechanism (Poison Pills)

**File:** `worklist/worklist.go`, lines 28-32

```go
func (w *Worklist) Finalize(numWorkers int) {
    for i := 0; i < numWorkers; i++ {
        w.Add(Entry{""})
    }
}
```

After all real files have been pushed, `Finalize` sends exactly `numWorkers` (10) entries with an empty `Path`. Each worker, upon receiving an empty path, knows it's time to exit. This is a classic **poison pill** pattern:

- There are 10 workers.
- Each worker pulls exactly one poison pill and exits.
- Therefore, exactly 10 poison pills are needed.

**Why not just close the channel?** You could close the `jobs` channel instead, and workers would detect it with `job, ok := <-w.jobs`. The poison pill approach is an alternative that's more explicit and doesn't require changing the `Next()` API to return a second value.

---

### Phase 4: Worker Goroutines (Consumers)

**File:** `mgrep/main.go`, lines 52-71

```go
for i := 0; i < numWorkers; i++ {
    workersWg.Add(1)
    go func() {
        defer workersWg.Done()
        for {
            workEntry := wl.Next()
            if workEntry.Path != "" {
                workerResult := worker.FindInFile(workEntry.Path, args.SearchTerm)
                if workerResult != nil {
                    for _, r := range workerResult.Inner {
                        results <- r
                    }
                }
            } else {
                return
            }
        }
    }()
}
```

This loop spawns **10 worker goroutines**. Each one does the same thing:

1. **`workersWg.Add(1)`** -- register with the WaitGroup before launching.
2. **`go func() { ... }()`** -- spawn the goroutine.
3. **Infinite loop:**
   - **`wl.Next()`** -- pull the next job from the worklist. This **blocks** if the channel is empty (the goroutine sleeps until a job is available -- no busy-waiting, no CPU waste).
   - **If `Path` is not empty** -- it's a real file. Call `worker.FindInFile(...)` to search it.
     - If matches were found (`workerResult != nil`), iterate over every individual `Result` and push it into the `results` channel.
   - **If `Path` is empty** -- it's a poison pill. `return` exits the goroutine, and `defer workersWg.Done()` fires automatically.

**Key insight:** All 10 workers pull from the **same** channel. Go channels are safe for multiple concurrent readers -- each job is delivered to exactly one worker. This is automatic load balancing: whichever worker finishes first gets the next job.

#### How `wl.Next()` works

**File:** `worklist/worklist.go`, lines 15-18

```go
func (w *Worklist) Next() Entry {
    j := <-w.jobs
    return j
}
```

This is a **blocking receive** on the `jobs` channel. If the channel is empty, the goroutine calling `Next()` will **sleep** until a value is available. There is no spinning, no polling -- the Go runtime scheduler handles waking the goroutine up.

#### How `worker.FindInFile` works

**File:** `worker/worker.go`, lines 24-46

```go
func FindInFile(path string, find string) *Results {
    file, err := os.Open(path)
    if err != nil {
        fmt.Println("Error:", err)
        return nil
    }
    results := Results{make([]Result, 0)}
    scanner := bufio.NewScanner(file)
    lineNum := 1

    for scanner.Scan() {
        if strings.Contains(scanner.Text(), find) {
            r := NewResult(scanner.Text(), lineNum, path)
            results.Inner = append(results.Inner, r)
        }
        lineNum += 1
    }
    if len(results.Inner) == 0 {
        return nil
    } else {
        return &results
    }
}
```

**Step-by-step:**

1. **Open the file** with `os.Open`. If it fails (file deleted, permissions, etc.), print an error and return `nil`.
2. **Create a `bufio.Scanner`** to read the file line by line -- memory efficient, doesn't load the entire file.
3. **For each line** (`scanner.Scan()` returns `true` until EOF):
   - Check if the line **contains** the search term using `strings.Contains` (case-sensitive substring match).
   - If it matches, create a `Result` with the line text, line number, and file path, and append it to the results.
   - Increment `lineNum` regardless.
4. **Return:** If no matches were found, return `nil`. Otherwise, return a pointer to the `Results` struct.

**Note:** The file is never explicitly closed (`file.Close()` is missing). This is a minor resource leak -- in a long-running program this would be a bug, but since `FindInFile` returns quickly and the OS reclaims file handles when the process exits, it works in practice. If you're debugging file handle exhaustion, **this is the place to look**.

---

### Phase 5: The Display Goroutine (Printer)

**File:** `mgrep/main.go`, lines 73-94

```go
blockWorkersWg := make(chan struct{})
go func() {
    workersWg.Wait()
    close(blockWorkersWg)
}()

var displayWg sync.WaitGroup
displayWg.Add(1)

go func() {
    for {
        select {
        case r := <-results:
            fmt.Printf("%v[%v]:%v\n", r.Path, r.LineNum, r.Line)
        case <-blockWorkersWg:
            if len(results) == 0 {
                displayWg.Done()
                return
            }
        }
    }
}()
```

This is the most nuanced part of the program. There are **two** goroutines launched here:

#### Goroutine A: The "Workers Done" Signal (lines 73-77)

```go
blockWorkersWg := make(chan struct{})
go func() {
    workersWg.Wait()
    close(blockWorkersWg)
}()
```

- Creates an **unbuffered, empty-struct channel** called `blockWorkersWg`.
- Launches a goroutine that **blocks** on `workersWg.Wait()` -- it sleeps until the discovery goroutine + all 10 workers have called `Done()`.
- Once they're all done, it **closes** the channel.

**Why this pattern?** You cannot use `workersWg.Wait()` directly inside a `select` statement. The `select` statement only works with channel operations. So this goroutine converts the WaitGroup's "done" signal into a channel close, which *can* be used in `select`.

A closed channel is special in Go: **every receive on a closed channel returns immediately** with the zero value. So `<-blockWorkersWg` will block until the channel is closed, and then it will succeed on every subsequent attempt.

#### Goroutine B: The Display Loop (lines 82-94)

```go
go func() {
    for {
        select {
        case r := <-results:
            fmt.Printf("%v[%v]:%v\n", r.Path, r.LineNum, r.Line)
        case <-blockWorkersWg:
            if len(results) == 0 {
                displayWg.Done()
                return
            }
        }
    }
}()
```

This runs an infinite `select` loop with two cases:

| Case | Trigger | Action |
|------|---------|--------|
| `case r := <-results` | A result is available in the results channel | Print it in the format `path[lineNum]:line` |
| `case <-blockWorkersWg` | All workers are done (channel closed) | Check if there are any remaining results. If not, mark display as done and exit. If there are, loop again to drain them. |

**Why the `len(results) == 0` check?** When `blockWorkersWg` is closed, `select` might pick the second case even if there are still results in the channel buffer. The `len` check ensures we don't exit early -- we only exit when the workers are done **AND** all results have been printed.

**Output format:** `filepath[lineNumber]:matching line content`

---

### Phase 6: Waiting for Completion

**File:** `mgrep/main.go`, line 96

```go
displayWg.Wait()
```

The main goroutine blocks here until `displayWg.Done()` is called by the display goroutine. This ensures the program doesn't exit until every match has been printed to the terminal.

**Without this line,** `main()` would return immediately after spawning all the goroutines, and the process would exit before any work is done.

---

## 5. The Worklist Package -- Thread-Safe Job Queue

**File:** `worklist/worklist.go` (complete)

```go
package worklist

type Entry struct {
    Path string
}

type Worklist struct {
    jobs chan Entry
}

func (w *Worklist) Add(work Entry) {
    w.jobs <- work
}

func (w *Worklist) Next() Entry {
    j := <-w.jobs
    return j
}

func New(buffSize int) Worklist {
    return Worklist{make(chan Entry, buffSize)}
}

func NewJob(path string) Entry {
    return Entry{path}
}

func (w *Worklist) Finalize(numWorkers int) {
    for i := 0; i < numWorkers; i++ {
        w.Add(Entry{""})
    }
}
```

| Function | What It Does |
|----------|-------------|
| `New(buffSize)` | Creates a new `Worklist` with a buffered channel of the given size |
| `NewJob(path)` | Convenience constructor for an `Entry` |
| `Add(work)` | Pushes a job into the channel. **Blocks** if the buffer is full (backpressure). |
| `Next()` | Pulls a job from the channel. **Blocks** if the buffer is empty (waits for work). |
| `Finalize(n)` | Sends `n` empty entries (poison pills) to signal workers to stop |

**The entire package is thread-safe because Go channels are inherently safe for concurrent access.** No mutexes, no locks -- the channel handles all synchronization internally.

---

## 6. The Worker Package -- File Searching Logic

**File:** `worker/worker.go` (complete)

```go
package worker

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

type Result struct {
    Line    string
    LineNum int
    Path    string
}

type Results struct {
    Inner []Result
}

func NewResult(line string, lineNum int, path string) Result {
    return Result{line, lineNum, path}
}

func FindInFile(path string, find string) *Results {
    file, err := os.Open(path)
    if err != nil {
        fmt.Println("Error:", err)
        return nil
    }
    results := Results{make([]Result, 0)}
    scanner := bufio.NewScanner(file)
    lineNum := 1

    for scanner.Scan() {
        if strings.Contains(scanner.Text(), find) {
            r := NewResult(scanner.Text(), lineNum, path)
            results.Inner = append(results.Inner, r)
        }
        lineNum += 1
    }
    if len(results.Inner) == 0 {
        return nil
    } else {
        return &results
    }
}
```

This package has **zero concurrency awareness**. It's a pure function: give it a file path and a search term, and it returns all matching lines. This clean separation means you can test `FindInFile` without any goroutines.

---

## 7. Why Goroutines? What Would Happen Without Them?

### The Sequential Alternative

Without goroutines, the program would look like this:

```go
func main() {
    // 1. Walk ALL directories first (must finish before any searching begins)
    files := discoverAllFiles(args.SearchDir)

    // 2. Search each file ONE AT A TIME
    for _, file := range files {
        result := worker.FindInFile(file, args.SearchTerm)
        if result != nil {
            for _, r := range result.Inner {
                fmt.Printf("%v[%v]:%v\n", r.Path, r.LineNum, r.Line)
            }
        }
    }
}
```

### Why This Is Slower

| Aspect | Sequential | With Goroutines |
|--------|-----------|----------------|
| **Directory discovery** | Must complete before ANY file is searched | Runs concurrently -- workers start searching files as soon as the first file is discovered |
| **File I/O** | One file at a time. While waiting for disk, CPU is idle | 10 files read in parallel. While one goroutine waits for disk I/O, others are actively searching |
| **CPU utilization** | Single core | Go scheduler distributes goroutines across all available CPU cores |
| **Pipeline** | No pipeline -- each stage waits for the previous one | Three stages overlap: discovering, searching, and printing happen simultaneously |
| **Latency to first result** | Must walk ALL directories, then start searching the first file | First result appears as soon as one file in one directory is found and searched |

### Concrete Example

Imagine searching 10,000 files across 500 directories:

- **Sequential:** Walk 500 dirs (say 2 seconds) + search 10,000 files one by one (say 30 seconds) = **32 seconds total**, first result after ~2 seconds.
- **Concurrent (this program):** Discovery starts pushing files immediately. 10 workers start searching as files arrive. Discovery, searching, and printing overlap. Total time might be **~5-6 seconds**, first result within milliseconds.

### Why Specifically 10 Workers?

The number 10 is a reasonable default for I/O-bound work. File searching is mostly limited by disk speed, not CPU. With 10 goroutines, the OS can queue up multiple read requests, keeping the disk busy. Going much higher (e.g., 1000) would waste memory on goroutine stacks and potentially thrash the disk with too many concurrent reads.

---

## 8. Synchronization Deep Dive

There are **three synchronization mechanisms** in this program:

### 1. `workersWg` (sync.WaitGroup)

**Tracks:** 1 discovery goroutine + 10 worker goroutines = 11 total `Add` calls.

```
Add(1)  -- discovery goroutine     (line 44)
Add(1)  -- worker goroutine 1      (line 53, i=0)
Add(1)  -- worker goroutine 2      (line 53, i=1)
...
Add(1)  -- worker goroutine 10     (line 53, i=9)
```

Each goroutine calls `Done()` via `defer` when it exits. When the counter reaches 0, `workersWg.Wait()` unblocks.

### 2. `blockWorkersWg` (chan struct{})

Converts the WaitGroup signal into a channel operation so it can be used in `select`. This is a standard Go pattern.

### 3. `displayWg` (sync.WaitGroup)

**Tracks:** Just 1 goroutine -- the display goroutine. The main goroutine blocks on `displayWg.Wait()` to ensure all output is printed before the program exits.

### Complete Shutdown Sequence

```
1. discoverDirs() finishes walking all directories
2. Finalize() sends 10 poison pills into the jobs channel
3. Each worker receives a poison pill and exits (workersWg.Done())
4. Discovery goroutine exits (workersWg.Done())
5. workersWg counter reaches 0
6. workersWg.Wait() unblocks in the signal goroutine
7. blockWorkersWg channel is closed
8. Display goroutine's select picks up the closed channel signal
9. Display goroutine drains any remaining results
10. Display goroutine calls displayWg.Done() and exits
11. displayWg.Wait() in main() unblocks
12. main() returns, program exits
```

---

## 9. Debugging Guide

### Common Issues and Where to Look

| Symptom | Likely Cause | Where to Look |
|---------|-------------|---------------|
| Program hangs forever | Deadlock -- a goroutine is blocked on a channel that will never receive | Check if `Finalize` is called with the correct `numWorkers` count |
| No output at all | Search term not found, or `SearchDir` is wrong | Add `fmt.Println` in `discoverDirs` to verify files are being found |
| Missing results | Race condition in results channel or display exits too early | Check the `len(results) == 0` guard in the display goroutine |
| "Error reading directory" | Invalid `SearchDir` path or permissions | Check `args.SearchDir` value after parsing |
| "Error: open ..." messages | File was deleted between discovery and search, or permissions | Check `worker.FindInFile` error handling |
| Too many open files (OS error) | No `file.Close()` in `FindInFile` | Add `defer file.Close()` after `os.Open` in `worker/worker.go` line 25 |
| Panics with "send on closed channel" | Indicates a logic error in channel management | Channels in this codebase are never explicitly closed except `blockWorkersWg` -- check if any code was added that closes `results` or `jobs` |

### Adding Debug Logging

To trace goroutine lifecycle:

```go
// In each worker goroutine (mgrep/main.go, line 54)
go func(id int) {
    fmt.Printf("[worker %d] started\n", id)
    defer fmt.Printf("[worker %d] exited\n", id)
    defer workersWg.Done()
    // ... rest of worker logic
}(i)
```

To trace job flow:

```go
// In worklist.Add (worklist/worklist.go, line 11)
func (w *Worklist) Add(work Entry) {
    fmt.Printf("[worklist] adding: %s\n", work.Path)
    w.jobs <- work
}
```

### Using Go's Race Detector

Run with the built-in race detector to catch data races:

```bash
go run -race ./mgrep "search_term" "./search_dir"
```

This will report any concurrent reads/writes to shared memory that are not properly synchronized.

---

## 10. How to Run

```bash
# Install dependencies
go mod download

# Run directly
go run ./mgrep <search_term> <search_dir>

# Examples
go run ./mgrep "func" "."
go run ./mgrep "TODO" "./src"

# Build a binary
go build -o mgrep.exe ./mgrep
./mgrep.exe "func" "."

# Run with race detector (for debugging)
go run -race ./mgrep "func" "."
```
