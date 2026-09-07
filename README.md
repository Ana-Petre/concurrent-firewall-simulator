# Parallel Packet Firewall

A multithreaded packet firewall implemented in C, built around a shared **ring buffer** and a **producer-consumer** architecture. A single producer thread generates simulated network packets and pushes them into a fixed-size circular buffer; multiple consumer threads pull packets from the buffer, apply filtering logic, and log the accept/drop decision for each one — in the correct chronological order, even though the consumers finish their work non-deterministically.

## Overview

Instead of processing real network traffic, the firewall works with synthetic packets made up of:

- a source (number)
- a destination (number)
- a timestamp (number)
- a payload

A producer thread creates these packets and inserts them into a shared circular buffer. Consumer threads (the actual firewall workers) pick packets from the buffer, decide whether to **PASS** or **DROP** them, and write the result to a log file.


## Acknowledgments

This is a solution to the **"Parallel Firewall"** assignment from the Operating Systems course at the **National University of Science and Technology POLITEHNICA Bucharest (UPB)**. The assignment statement, skeleton code, and automated grading/testing infrastructure (`tests/`, `utils/`, `src/serial.c`, `src/packet.c`) were provided by the course staff. Only the ring buffer implementation and consumer thread logic (`src/consumer.c` and related synchronization code) are my own work.

---

*This project was completed as part of an Operating Systems coursework assignment.*

## Key Features

- **Custom ring (circular) buffer** implemented from scratch, with proper synchronization for concurrent access.
- **Multiple consumer threads** processing packets in parallel — no busy waiting, no polling loops. Threads block and are woken up via condition variables/semaphores.
- **Deterministic, timestamp-ordered logging**: even though consumers process packets out of order, the log file is written incrementally, sorted by packet timestamp — without a post-processing sort step.
- **Graceful shutdown**: consumer threads exit cleanly once the producer signals there are no more packets coming.

## Architecture

```
Producer Thread                Ring Buffer                 Consumer Threads
┌──────────────┐          ┌───────────────────┐          ┌──────────────────┐
│  Generates    │  enqueue │  Fixed-size        │ dequeue  │  Apply filter     │
│  packets      │ ───────> │  circular array    │ ───────> │  Log PASS/DROP    │
│               │          │  + sync primitives │          │  (ordered by ts)  │
└──────────────┘          └───────────────────┘          └──────────────────┘
```

### Ring Buffer Interface

| Function                | Purpose                                             |
|--------------------------|------------------------------------------------------|
| `ring_buffer_init()`     | Allocates the buffer and synchronization primitives |
| `ring_buffer_enqueue()`  | Adds a packet to the buffer (producer side)         |
| `ring_buffer_dequeue()`  | Removes a packet from the buffer (consumer side)    |
| `ring_buffer_stop()`     | Signals a thread is done using the buffer           |
| `ring_buffer_destroy()`  | Frees all buffer resources                          |

## Project Structure

```
.
├── src/
│   ├── serial.c       # Reference single-threaded implementation
│   ├── consumer.c      # Consumer thread logic (main implementation work)
│   ├── packet.c        # Packet parsing/filtering (provided)
│   └── ...             # Ring buffer + firewall skeleton
├── utils/               # Debugging and logging helpers
└── tests/
    ├── in/              # Generated test inputs
    └── grade.sh         # Automated grading script
```

## Building

```bash
cd src/
make
```

This produces two binaries: `serial` (reference implementation) and `firewall` (the parallel version).

## Running

```bash
./firewall <input_file> <output_file> <number_of_consumers>
```

Example:

```bash
./firewall ../tests/in/test_1000.in output.log 4
```

The output of the parallel `firewall` must match the output of `serial` for the same input.

## Testing

The project includes an automated checker:

```bash
cd tests/
make check      # generate and run all tests
./grade.sh       # run the grading script
make lint        # run style linters (checkpatch.pl, cpplint, shellcheck)
make distclean   # remove generated test files
```

Grading breakdown:

- **10 pts** — single-consumer solution with a working ring buffer
- **50 pts** — correct multi-consumer solution
- **30 pts** — multi-consumer solution with correctly timestamp-ordered logs
- **10 pts** — code style (linters)

## Synchronization Constraints

- Threads must **block/yield**, not busy-wait, while the buffer is empty or full.
- Logs must be written **as packets are processed**, already in ascending timestamp order — sorting after the fact is not allowed.
- At least `num_consumers + 1` threads must be running at all times (consumers + producer).

## Tech Stack

- C
- POSIX threads (`pthread`) — mutexes, condition variables / semaphores


