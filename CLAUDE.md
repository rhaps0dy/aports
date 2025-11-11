# Building Alpine Linux with Fil-C

## Overview
We're rebuilding Alpine Linux packages to use Fil-C's clang compiler with InvisiCap pointer representation. This provides spatial memory safety for C code.

## Fil-C Architecture: Two-Libc Sandwich
Fil-C uses a unique architecture with two C libraries:
- **User libc (musl)**: Standard C library that applications link against
- **Pizlo runtime (libpizlo.so)**: Sits between user libc and yolo libc, handles InvisiCap pointer operations
- **Yolo libc (libyoloc.so)**: Low-level C library used by the runtime

Programs compiled with Fil-C link against both the standard musl and the Fil-C runtime libraries.

## Current Status

### ✅ Completed: musl (1.2.5-r21)
Location: `/workspace/aports/main/musl/`

Successfully built musl C library with Fil-C clang. Key modifications:
- Use `/workspace/fil-c/build/bin/clang` as CC
- Add `-B/usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/` to find GCC runtime files (crtbeginS.o, libgcc.a, etc.)
- Use `/workspace/fil-c/build/bin/llvm-ranlib` as RANLIB
- Build `libssp_nonshared.a` from `__stack_chk_fail_local.c`
- Create `/lib` directory before installing libc.so
- Link `/lib/libc.so` → `/lib/libc.musl-x86_64.so.1`

Packages created:
- `musl-1.2.5-r21.apk` - main library
- `musl-dev-1.2.5-r21.apk` - headers and development files
- `musl-utils-1.2.5-r21.apk` - utilities (getconf, getent, iconv, ldd)
- `musl-dbg-1.2.5-r21.apk` - debug symbols
- `musl-libintl-1.2.5-r21.apk` - libintl header

### ✅ Completed: fil-c-runtime (0.674-r0)
Location: `/workspace/aports/main/fil-c-runtime/`

Package containing Fil-C runtime libraries that musl depends on:
- `libpizlo.so` - Fil-C runtime library
- `libyoloc.so` - Yolo libc (low-level C library for Fil-C)

Both musl and musl-utils packages depend on this via `so:libpizlo.so` and `so:libyoloc.so`.

**IMPORTANT**: These libraries are installed to `/usr/lib/` by the package system, but at build time they're found at `/workspace/fil-c/pizfix/lib/`. This is expected - the libraries aren't standard and show errors with `ldd` because they're part of the Fil-C two-libc architecture.

### ✅ Completed: OpenSSL (3.3.1) - Alpine Package
Location: `/workspace/aports/main/openssl-filc/`

Successfully packaged OpenSSL 3.3.1 with Fil-C patches as an Alpine package. This was required for busybox's ssl_client utility.

**Key Challenge**: The Fil-C patches from the fil-c repo were designed for OpenSSL 3.3.1 specifically. Initially attempted to use Alpine's OpenSSL 3.5.4, but downgraded to 3.3.1 for compatibility.

**Extracting Fil-C Patches**: CRITICAL: When extracting patches from the fil-c repo, you MUST restrict the git diff to the specific project directory to avoid including the entire monorepo:
```bash
cd /workspace/fil-c && git diff <base_commit>..HEAD -- projects/openssl-3.3.1 | sed 's|projects/openssl-3.3.1/||g' > patch.patch
```
Without the `-- projects/openssl-3.3.1` restriction, the diff will take forever and be gigabytes in size. The `sed` command strips the path prefix so patches apply correctly in the Alpine build.

**Build Process**:
1. Extract Fil-C patch using git diff (see command above)
2. Create APKBUILD with patches: `openssl-3.3.1-filc.patch` (Fil-C changes) and `auxv.patch` (Alpine-specific)
3. Build with `abuild -r` which automatically:
   - Downloads OpenSSL 3.3.1 source
   - Applies both patches
   - Configures with Fil-C compiler
   - Builds shared libraries only (`make build_sw`)
   - Creates .apk packages

**Understanding "pizlonated_" Prefix**:
During investigation, we discovered that the "pizlonated_" prefix is **Fil-C compiler name mangling** for functions that interface with unsafe assembly code. The Fil-C patches add C forwarder functions like:
```c
void sha1_block_data_order(void *c, const void *p, size_t len) {
    zcheck(c, sizeof(SHA_CTX));
    zcheck_readonly(p, zchecked_mul(len, SHA_CBLOCK));
    zunsafe_buf_call(zchecked_mul(len, SHA_CBLOCK), "sha1_block_data_order", c, p, len);
}
```

The Fil-C compiler automatically adds the "pizlonated_" prefix to these when compiling, creating symbols like `pizlonated_sha1_block_data_order`. This is not in the fil-c source code - it's generated at compile time.

**Fil-C Patches Overview**:
- Add `*_asm_forward.c` files for assembly functions (AES, SHA, etc.)
- These forwarders use `zcheck()`, `zunsafe_call()`, `zunsafe_buf_call()` etc. from `<stdfil.h>`
- Modify `build.info` files to include forwarders alongside assembly files
- Disable inline assembly for Fil-C (`!defined(__FILC__)` conditions)
- Export unsafe symbols with `.filc_unsafe_export` assembly directive

**Packages Created**:
- `openssl-filc-3.3.1-r0.apk` (2.1MB) - Main package with openssl command-line tool
- `libcrypto3-filc-3.3.1-r0.apk` (8.9MB) - Cryptography library
- `libssl3-filc-3.3.1-r0.apk` (2.4MB) - SSL/TLS library
- `openssl-filc-dev-3.3.1-r0.apk` (351KB) - Development headers and .so symlinks
- `openssl-filc-libs-static-3.3.1-r0.apk` (23.9MB) - Static libraries
- `openssl-filc-dbg-3.3.1-r0.apk` (8.0MB) - Debug symbols

**ABI Compatibility Note**: OpenSSL built with Fil-C is NOT ABI-compatible with standard OpenSSL. Any programs linking against it must also be built with Fil-C. The packages correctly conflict with the system OpenSSL packages to prevent installation conflicts.

### ✅ Completed: busybox (1.37.0-r24)
Location: `/workspace/aports/main/busybox/`

Successfully built busybox with Fil-C compiler. Key modification: ssl_client utility now links against Fil-C OpenSSL.

**Bootstrap Challenge**: Cannot use `openssl-filc-dev` as a build dependency because it conflicts with system OpenSSL (needed by abuild, apk-tools, python3, etc.).

**Solution**: Modified APKBUILD to directly reference Fil-C OpenSSL libraries at build time:
```bash
# In build() function for ssl_client:
${CC:-${CROSS_COMPILE}gcc} $CPPFLAGS $CFLAGS \
    -I/workspace/openssl-3.5.4-filc/include \
    "$srcdir"/ssl_client.c -o "$_dyndir"/ssl_client $LDFLAGS \
    -L/workspace/openssl-3.5.4-filc -lssl -lcrypto \
    -Wl,-rpath,/workspace/openssl-3.5.4-filc
```

**Packages Created**:
- `busybox-1.37.0-r24.apk` - Main busybox binary (808KB)
- `busybox-static-1.37.0-r24.apk` - Static build with internal SSL (1MB)
- `ssl_client-1.37.0-r24.apk` - Standalone SSL client linked to Fil-C OpenSSL
- Plus binsh, suid, openrc, extras subpackages

**Note**: The main busybox binary links only to musl (which links to libpizlo/libyoloc). Only ssl_client directly links to OpenSSL.

## Build Requirements

### Key Compiler Flags for Fil-C
- **CC**: `/workspace/fil-c/build/bin/clang`
- **RANLIB**: `/workspace/fil-c/build/bin/llvm-ranlib`
- **-B/usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/**: Tells clang where to find GCC runtime files (crtbeginS.o, crtendS.o, libgcc.a, libgcc_eh.a)
  - These are CRT (C runtime) startup/shutdown objects and low-level GCC runtime functions
  - Only needed at **build time**, not at runtime
  - Files come from the system `gcc` package

### Why -B Flag is Needed
Clang needs to find GCC-specific runtime files during linking:
1. CRT objects (crtbeginS.o, crtendS.o) for C program startup/shutdown
2. libgcc.a for low-level functions (64-bit division on 32-bit, etc.)
3. libgcc_eh.a for exception handling support

Without `-B`, clang looks in the wrong place and linking fails.

## Bootstrap Strategy

### Approach: Build Core Packages First
1. ✅ musl - C library (RPATH issue - needs fix)
2. ✅ fil-c-runtime - Fil-C runtime libraries
3. ✅ openssl-filc - Cryptography library (required by busybox)
4. ✅ busybox - Essential Unix utilities (ssl_client with Fil-C OpenSSL)
5. ✅ alpine-baselayout - Directory structure (noarch package)
6. 🔄 Minimal rootfs - Creating bootable Fil-C Alpine environment
7. alpine-keys, apk-tools - Package manager (future)
8. linux kernel (future)

### Dependency Management
- Alpine's `abuild -r` handles recursive dependency building
- Each package's APKBUILD lists dependencies
- Build packages in waves based on dependency depth

### Common Modifications Needed
Most packages will likely need:
1. Set `CC=/workspace/fil-c/build/bin/clang`
2. Add `-B/usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/` to CFLAGS
3. Set `RANLIB=/workspace/fil-c/build/bin/llvm-ranlib`
4. Ensure fil-c-runtime is installed first

## Testing
Built packages are in: `/home/dev/packages/main/x86_64/`

To install:
```bash
sudo apk add --allow-untrusted /home/dev/packages/main/x86_64/<package>.apk
```

Repository index is automatically updated by abuild.

### 🔄 In Progress: Minimal Fil-C Alpine Rootfs
Location: `/workspace/filc-alpine-rootfs/`

Created a minimal rootfs with core Fil-C packages:
- alpine-baselayout-data + alpine-baselayout
- fil-c-runtime (libpizlo.so, libyoloc.so)
- musl (Fil-C build)
- busybox + busybox-binsh
- libcrypto3-filc + libssl3-filc
- ssl_client

**Critical RPATH Issue**: musl's RUNPATH includes build directory first:
```
RUNPATH: /workspace/fil-c/build/bin/../../pizfix/lib:/usr/lib
```

This causes issues when trying to run binaries in chroot because:
1. The build directory path is checked first
2. May load wrong versions of libpizlo/libyoloc
3. Results in segfaults when libraries mismatch

**Current Status**:
- Packages install successfully to rootfs
- `ld-musl-x86_64.so.1` symlink created manually
- chroot attempts result in segmentation fault
- Need to either:
  1. Remove build path from RUNPATH entirely, OR
  2. Rebuild musl without build-time RUNPATH, OR
  3. Use patchelf to fix RUNPATH after build

## Known Issues / Notes
- **CRITICAL**: musl RUNPATH includes build directory which causes runtime issues
- The `-B` flag path won't be in a base system - we may need to package GCC runtime files or configure clang differently for production
- Fil-C libraries show errors with standard tools like `ldd` because they use InvisiCap pointer representation - this is expected
- `ld-musl-x86_64.so.1` symlink must be created manually in rootfs (not created by musl package)

## Next Steps
1. **Fix musl RUNPATH issue** (CRITICAL)
   - Remove build directory from RUNPATH
   - Options: Modify configure flags, use patchelf post-build, or set proper LDFLAGS
   - Must ensure runtime only looks in `/usr/lib` for libpizlo/libyoloc
2. **Complete working Fil-C Alpine rootfs**
   - Fix `ld-musl-x86_64.so.1` symlink creation in musl package
   - Test chroot functionality
   - Create tarball for Docker/container use
3. **Build additional core packages**
   - apk-tools (may need Fil-C build for full Fil-C system)
   - alpine-keys
   - Additional utilities as needed

## Workflow for Future Packages
For packages with Fil-C patches in the fil-c repo:
1. Find base commit: `cd /workspace/fil-c/projects/<package> && git blame <file> | head -1 | awk '{print $1}'`
2. Extract patch: `cd /workspace/fil-c && git diff <commit>..HEAD -- projects/<package> | sed 's|projects/<package>/||g' > <package>-filc.patch`
3. Create APKBUILD in `/workspace/aports/main/<package>-filc/` or modify existing
4. Add Fil-C compiler flags and patch to sources
5. Build with `abuild -r`
6. Commit to git regularly
