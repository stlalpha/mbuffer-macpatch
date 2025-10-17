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


===============================================================================

Patch 2: Fix idev.so and tapetest.so Compilation on macOS
==========================================================

This patch fixes compilation failures for the optional debugging modules
(idev.so and tapetest.so) on macOS.

PROBLEM:
--------
The configure script uses objdump to detect C library symbol names, but
objdump doesn't work with macOS's Mach-O binary format. This leaves
LIBC_OPEN, LIBC_READ, LIBC_WRITE, and LIBC_FSTAT macros empty in config.h,
causing compilation errors when the preprocessor expands function names.

Example error:
  idev.c:55:15: error: expected identifier or '('
  int LIBC_OPEN(const char *path, int oflag, ...)
                ^

SOLUTION:
---------
Add conditional detection and definition of C library symbols:
- On macOS: use single underscore prefix (_open, _read, _write, _fstat)
- On other platforms: use unprefixed names (open, read, write, fstat)

IMPLEMENTATION:
---------------
- idev.c: Added LIBC_OPEN, LIBC_READ checks with macOS fallbacks
- tapetest.c: Replaced #error with conditional macOS symbol definitions
- Both files: Use TOSTRING() macro for dlsym() to get correct symbol names

BENEFITS:
---------
- idev.so and tapetest.so now compile cleanly on macOS
- No impact on other platforms (fallback maintains existing behavior)
- Optional modules work correctly with LD_PRELOAD on macOS

FILES MODIFIED:
---------------
- idev.c: Added conditional symbol definitions
- tapetest.c: Added conditional symbol definitions and STRINGIFY macros

PATCH APPLICATION:
------------------
To apply this patch:
  git apply 0002-fix-idev-tapetest-macos.patch

Or with patch command:
  patch -p1 < 0002-fix-idev-tapetest-macos.patch

NOTE:
-----
This patch can be applied independently or together with patch 0001.
The idev.so and tapetest.so modules are optional debugging tools and
not required for normal mbuffer operation.

Generated: 2025-10-17


===============================================================================

Patch 3: Complete macOS Support (Combined Patch)
=================================================

This is a single comprehensive patch that combines all macOS compatibility
fixes and the 2GB buffer limit removal. It includes everything from patches
0001 and 0002, plus additional macOS-specific improvements.

WHAT'S INCLUDED:
----------------
1. Semaphore replacement with condition variables (removes 2GB limit)
2. macOS memory detection using Mach VM APIs
3. macOS fsync error handling (EOPNOTSUPP/ENOTSUP)
4. idev.so and tapetest.so macOS compilation fixes
5. Updated README documentation

PROBLEM:
--------
mbuffer would not compile or run on macOS due to:
- macOS doesn't implement sem_init() for unnamed semaphores
- objdump doesn't work with Mach-O binaries (breaks symbol detection)
- Missing macOS memory detection APIs
- 2GB buffer size limitation (32-bit semaphore counters)

SOLUTION:
---------
Complete rewrite of synchronization to use condition variables + 64-bit
counters, plus platform-specific code for macOS APIs.

BENEFITS:
---------
- Full macOS compatibility (compiles and runs on macOS)
- No buffer size limitations (supports >2GB buffers)
- All components work (mbuffer, idev.so, tapetest.so)
- Proper macOS memory detection (no warnings)
- Correct fsync error handling for APFS

FILES MODIFIED:
---------------
Core synchronization:
- globals.h/c: New condition variable sync primitives
- common.h/c: Counter operation functions
- mbuffer.c: Semaphore → condition variable conversion
- input.c: Semaphore → condition variable conversion
- settings.c: Removed semaphore limit checks

macOS-specific:
- mbuffer.c: Added Mach VM memory detection, fsync error handling
- idev.c: macOS C library symbol handling
- tapetest.c: macOS C library symbol handling

Documentation:
- README: Updated MacOS and 64 Bit Buffers sections

PATCH APPLICATION:
------------------
To apply this patch:
  git apply 0003-complete-macos-support-combined.patch

Or with patch command:
  patch -p1 < 0003-complete-macos-support-combined.patch

RECOMMENDATION:
---------------
Use this patch (0003) if you want complete macOS support in one go.
Or use patches 0001 and 0002 separately if you prefer incremental changes.

TESTING:
--------
Tested on macOS with:
- Buffers up to 6GB (beyond old 2GB limit)
- SMB mount I/O (real-world network buffering)
- All compilation warnings resolved
- idev.so and tapetest.so compile cleanly

Example test:
  dd if=/dev/zero bs=1m count=10000 | ./mbuffer -m 5G -v 3 -R 100m | dd of=/dev/null

Generated: 2025-10-17
