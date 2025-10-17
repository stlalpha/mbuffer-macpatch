Patch for mbuffer: Add macOS Support with Semaphore Wrapper
=============================================================

This patch enables mbuffer to build and run on macOS by working around
platform-specific limitations in POSIX semaphore support.

PROBLEM:
--------
mbuffer fails to build and run on macOS due to several platform limitations:

1. macOS does not support unnamed POSIX semaphores (sem_init/sem_destroy)
   - sem_init() returns ENOSYS (Function not implemented)

2. macOS does not provide sem_getvalue() even for named semaphores
   - The function exists but is deliberately not implemented

3. macOS does not have the same sysctl APIs for memory detection
   - Linux/FreeBSD/NetBSD APIs not available on macOS

4. macOS returns EOPNOTSUPP/ENOTSUP for fsync() on pipes/TTY
   - Previous error handling only checked EINVAL/EBADRQC

SOLUTION:
---------
Add a semaphore wrapper layer that:
- Uses named semaphores (sem_open) instead of unnamed semaphores on macOS
- Implements sem_getvalue() using atomic counters
- Adds macOS-specific memory detection using Mach VM APIs
- Expands fsync error handling for macOS errno values

IMPLEMENTATION:
---------------
1. Semaphore Wrapper (common.h/c):
   - Defines mbuffer_sem_t structure containing:
     * sem_t* pointer (for named semaphore)
     * atomic_int counter (for value tracking)
     * char name[32] (unique semaphore name)

   - Implements wrapper functions:
     * mbuffer_sem_init(): Creates named semaphore with unique name
     * mbuffer_sem_post(): Posts to semaphore and increments atomic counter
     * mbuffer_sem_wait(): Waits on semaphore and decrements atomic counter
     * mbuffer_sem_getvalue(): Returns atomic counter value
     * mbuffer_sem_destroy(): Closes and unlinks named semaphore

   - Uses preprocessor to replace sem_* with mbuffer_sem_* on affected platforms

2. Memory Detection (mbuffer.c):
   - Added __APPLE__ code path in initDefaults()
   - Uses host_statistics64() with HOST_VM_INFO64 to query VM stats
   - Calculates available memory from free + inactive + purgeable + speculative pages
   - Eliminates "no mechanism to determine number of available pages" warning on macOS

3. Error Handling (mbuffer.c):
   - Added EOPNOTSUPP and ENOTSUP to fsync error checks
   - Prevents spurious warnings when syncing pipes/stdout on macOS

4. Build System (configure.in):
   - Added AC_CHECK_FUNCS([sem_getvalue]) to detect sem_getvalue support
   - Sets HAVE_SEM_GETVALUE appropriately, triggering wrapper on macOS

BENEFITS:
---------
- mbuffer now builds and runs successfully on macOS
- Maintains full compatibility with existing Linux/FreeBSD/NetBSD/Solaris builds
- Thread-safe semaphore value tracking using C11 atomics
- Proper memory detection on macOS (no more warnings about available pages)
- Clean abstraction layer - no changes needed to core mbuffer logic

TESTING:
--------
Tested on macOS (Apple Silicon) with various buffer configurations:
  dd if=/dev/zero bs=1M count=100 | ./mbuffer -m 100M | wc -c
  dd if=/dev/zero bs=1M count=100 | ./mbuffer -m 500M | wc -c

All tests pass successfully. mbuffer operates correctly with the semaphore
wrapper providing transparent compatibility.

FILES MODIFIED:
---------------
- common.h: Added mbuffer_sem_t structure and wrapper function declarations
- common.c: Implemented semaphore wrapper functions (81 lines of new code)
- mbuffer.c: Added macOS memory detection and expanded fsync error handling
- globals.h: Added comment about include order for semaphore wrapper
- globals.c: Ensured common.h included before globals.h for wrapper definitions
- configure.in: Added sem_getvalue detection

TECHNICAL NOTES:
----------------
1. Semaphore Naming:
   - Names generated as: /mbuf_<PID>_<microseconds>_<counter>
   - Ensures uniqueness across multiple mbuffer instances
   - Old semaphores from crashed instances are automatically unlinked

2. Atomic Counters:
   - Uses C11 <stdatomic.h> for thread-safe value tracking
   - atomic_fetch_add/atomic_fetch_sub provide lock-free updates
   - atomic_load provides consistent reads without locks

3. Include Order:
   - common.h must be included before globals.h
   - Ensures sem_t typedef happens before globals.h declarations
   - Allows wrapper to completely replace standard semaphore type

4. Portability:
   - Wrapper only activated when HAVE_SEM_GETVALUE is not defined
   - All other platforms continue using standard POSIX semaphores
   - No performance impact on platforms with native sem_getvalue()

ALTERNATIVE APPROACHES:
-----------------------
This patch maintains the existing semaphore-based architecture with minimal
changes. For a more comprehensive solution that also removes the 2GB buffer
limitation, see the companion patch: 0001-condvar-replace-semaphores.patch

That alternative patch replaces semaphores entirely with condition variables
and 64-bit counters, which:
- Removes the sem_init() limitation (condition variables work on all platforms)
- Removes the 2GB buffer size limit (uses 64-bit counters instead of 32-bit semaphores)
- Provides better cross-platform compatibility

The choice between these approaches depends on maintainer preference:
- This patch: Minimal change, preserves semaphore architecture
- Condvar patch: More comprehensive, removes buffer size limitation

PATCH APPLICATION:
------------------
To apply this patch:
  git apply 0001-macos-semaphore-wrapper.patch

Or with patch command:
  patch -p1 < 0001-macos-semaphore-wrapper.patch

After applying, regenerate configure script:
  autoconf

Then build normally:
  ./configure
  make

COMPATIBILITY:
--------------
- Requires C11 for <stdatomic.h> support (all modern compilers)
- Uses standard POSIX named semaphores (sem_open) on macOS
- Maintains backward compatibility with all existing platforms
- No changes to command-line interface or user-facing behavior

Generated: 2025-10-17
