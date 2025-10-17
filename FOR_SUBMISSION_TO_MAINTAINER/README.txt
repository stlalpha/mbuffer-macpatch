Patch for mbuffer: Remove 2GB Buffer Size Limitation
=====================================================

This patch addresses the "64 Bit Buffers" limitation documented in the README.

PROBLEM:
--------
mbuffer's buffer size was limited to sysconf(_SC_SEM_VALUE_MAX) * Blocksize
(typically ~2GB) because POSIX semaphores use 32-bit integer counters.

SOLUTION:
---------
Replace POSIX semaphores with pthread condition variables and 64-bit counters.

IMPLEMENTATION:
---------------
- Replaced sem_t Dev2Buf and Buf2Dev with volatile long long counters
- Added pthread mutexes and condition variables for synchronization
- Created helper functions: counter_init(), counter_wait(), counter_post(), counter_getvalue()
- Updated all semaphore operations throughout the codebase
- Removed maxSemValue() limitation checks

BENEFITS:
---------
- Buffer size now only limited by available system memory
- Enables buffers larger than 2GB on systems with sufficient RAM
- Maintains thread-safe synchronization using mutex-protected 64-bit counters
- More portable (condition variables are better supported than unnamed semaphores)

TESTING:
--------
Tested on macOS with buffers up to 500MB. The implementation compiles cleanly
and passes basic functionality tests:
  dd if=/dev/zero bs=1M count=100 | ./mbuffer -m 500M | wc -c

FILES MODIFIED:
---------------
- globals.h/c: Defined new sync primitives and counters
- common.h/c: Implemented counter operation functions
- mbuffer.c: Replaced all semaphore operations in main and output threads
- input.c: Replaced all semaphore operations in input thread
- settings.c: Removed semaphore limit validation
- README: Updated documentation to reflect removed limitation

PATCH APPLICATION:
------------------
To apply this patch:
  git apply 0001-condvar-replace-semaphores.patch

Or with patch command:
  patch -p1 < 0001-condvar-replace-semaphores.patch

COMPATIBILITY:
--------------
- Requires C99 for 64-bit long long type
- Uses standard pthread mutexes and condition variables
- No platform-specific code added
- Maintains backward compatibility with existing command-line options

Generated: 2025-10-17
