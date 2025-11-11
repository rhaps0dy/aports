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

### ✅ Completed: OpenSSL (3.3.1)
Location: `/workspace/openssl-3.5.4-filc/`

Successfully built OpenSSL 3.3.1 with Fil-C clang. This was required for busybox's ssl_client utility.

**Key Challenge**: The Fil-C patches from the fil-c repo (openssl.patch) were designed for OpenSSL 3.3.1 specifically. Initially attempted to use Alpine's OpenSSL 3.5.4, but downgraded to 3.3.1 for compatibility.

**Build Process**:
1. Started with OpenSSL repo at tag `openssl-3.3.1`
2. Copied all patched files from `/workspace/fil-c/projects/openssl-3.3.1/` (which has Fil-C patches already committed in git)
3. Applied Alpine's `auxv.patch` for Alpine-specific fixes
4. Configured with: `CC="/workspace/fil-c/build/bin/clang -g -O2 -B/usr/lib/gcc/x86_64-alpine-linux-musl/14.2.0/" ./Configure linux-x86_64 enable-ktls shared no-ssl3 no-weak-ssl-ciphers --prefix=/usr --libdir=lib --openssldir=/etc/ssl`
5. Built successfully with `make -j16 build_sw`

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

**Built Libraries**:
- `libcrypto.so.3` (52.5MB) - Cryptography library
- `libssl.so.3` (15.4MB) - SSL/TLS library
- Static versions: `libcrypto.a` (77.2MB), `libssl.a` (22.2MB)

**ABI Compatibility Note**: OpenSSL built with Fil-C is NOT ABI-compatible with standard OpenSSL. Any programs linking against it must also be built with Fil-C.

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
1. ✅ musl - C library
2. ✅ OpenSSL - Cryptography library (required by busybox)
3. 🔄 busybox - Essential Unix utilities (next target)
4. alpine-baselayout - Directory structure
5. alpine-keys, apk-tools - Package manager
6. linux kernel

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

## Known Issues / Notes
- The `-B` flag path won't be in a base system - we may need to package GCC runtime files or configure clang differently for production
- Fil-C libraries show errors with standard tools like `ldd` because they use InvisiCap pointer representation - this is expected
- Runtime RPATH for libpizlo.so/libyoloc.so may need adjustment to prefer `/usr/lib` over build directory

## Next Steps
1. Create OpenSSL 3.3.1 package for Alpine (currently built but not packaged)
   - May need to create custom APKBUILD or modify existing one
   - Ensure proper dependencies on fil-c-runtime
2. Build busybox with Fil-C OpenSSL
   - Modify busybox APKBUILD to use Fil-C compiler and depend on Fil-C OpenSSL
   - The ssl_client utility should now compile successfully
3. Continue with remaining core packages
