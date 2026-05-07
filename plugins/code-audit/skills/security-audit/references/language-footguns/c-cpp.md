# C / C++ Security Footguns

## Scope

C and C++ — both raw and within larger systems (kernel modules, embedded, native libraries called from higher-level languages, performance-critical paths). C/C++ has the largest attack surface among mainstream languages because the language itself doesn't enforce memory safety.

The bulk of CVEs across the industry come from memory-safety issues in C/C++ codebases (browsers, kernels, libraries). Detecting all of them in a manual audit is impractical; the goal is to find systemic issues, identify high-risk patterns, and recommend tooling for the mechanical detection that humans can't do reliably.

## Memory Safety

### Buffer Overflows / Out-of-Bounds Writes

- **`strcpy(dest, src)`** — No bounds check; writes past end if `src` exceeds `dest` capacity. Use `strncpy` (with size; still beware non-null-termination) or `strlcpy` (BSD; safer) or `snprintf`.
- **`strcat`, `sprintf`, `gets`** — Same. `gets` is removed in C11 / C++14; if found, the codebase is very old.
- **`memcpy(dst, src, n)`** with `n` derived from user input without bounds check.
- **Array indexing without bounds check** — `arr[user_index]` where `user_index` is unbounded.
- **Off-by-one** — Looping `for (i = 0; i <= n; i++)` instead of `< n`; common in C-string handling.

### Out-of-Bounds Reads

- **`memcpy(dst, src, n)`** reading past `src` end — Heartbleed-class.
- **`strlen(buf)`** on non-null-terminated buffer — Reads until first null, possibly past buffer.
- **Format-string mismatches** — Reading args that aren't pushed (covered below).

### Use-After-Free

- Pointer used after `free`/`delete`. Common pattern: function returns pointer to local stack data; pointer to freed heap data.
- **`std::vector` invalidation** — Iterator invalidated after `push_back` causing reallocation; iterator deref is UAF.
- **Aliased ownership** — Two pointers to the same allocation; one frees, the other dereferences.

### Double-Free

- `free(p); free(p);` — Heap corruption; exploitable.
- Two pointers to same allocation, both freed.

### Memory Leaks

- Allocations never freed. Generally a reliability issue; security relevance: long-lived processes accumulate; potential DoS.
- Information disclosure: leaked memory may include sensitive data observable via debugging.

### Uninitialized Reads

- `int x; printf("%d", x);` — Reads garbage; if used in security decisions, undefined behavior.
- Stack-allocated structures partially initialized then sent on the wire — Information disclosure of stack contents.

## Format String Vulnerabilities

`printf`-family functions taking user input as the format string:

- `printf(userInput)` — Reads memory using `%s`, `%x`; writes memory using `%n`. Information disclosure and memory corruption.
- Always: `printf("%s", userInput)` — User input as argument, not format.
- Same applies to `fprintf`, `sprintf`, `snprintf`, `syslog`, custom `*printf` wrappers.

## Integer Issues

### Overflow / Underflow

- **Unsigned overflow** — Defined behavior in C/C++ (wraps), but produces unexpected values. Common bug:
  ```c
  size_t n = strlen(input) + 1;  // overflow if input is near SIZE_MAX
  buf = malloc(n);
  ```
- **Signed overflow** — Undefined behavior in C/C++; compilers may optimize away checks. Use signed-overflow-aware arithmetic (`__builtin_add_overflow`, `ckd_add` in C23, libraries like `Boost.SafeInt`).
- **Truncation** — Assigning `int` value to `short` or `char` truncates; loss of significant bits.
- **`malloc(n * size)` with `n` user-controlled** — Multiplication overflow; allocate small buffer, write large data.

### Sign Conversion

- `unsigned char c = -1;` — Becomes 255; comparisons across sign boundaries surprise.
- `int n; if (n > some_unsigned)` — Implicit conversion; -1 becomes a large unsigned.

## C String Handling

### Null Termination

- All C string functions assume null-terminated buffers. Failing to null-terminate after `strncpy` (which doesn't add a terminator if source fills destination) is a common bug.
- Buffer received from network may not be null-terminated; `strlen` past buffer end is UB.

### Wide Strings

- `wchar_t` size differs (2 bytes Windows, 4 bytes most Unix); cross-platform code surprises.
- `wcscpy`, `wcscat` — Same risks as narrow versions.

## C++ Specifics

### Object Lifetime

- **Returning reference/pointer to local** — Stack data destroyed on return; UAF.
- **`std::string_view` to a temporary** — Dangling.
- **Lambda captures** — Capture by reference of locals that go out of scope.
- **Iterator invalidation** — Container modification invalidates iterators in known patterns.

### Exception Safety

- **Unwinding through C boundaries** — Throwing through extern "C" function pointer is UB.
- **Resource leaks on throw** — Allocations not RAII-wrapped leak on exception.
- **Destructors throwing** — UB if during stack unwinding.

### `std::string` and `std::vector`

- `std::string::data()` returns pointer; if string mutated, pointer invalidated.
- `std::vector::data()` same.
- C++17 added size validation on some operations; older code may not.

### Smart Pointers

- `std::unique_ptr` — Ownership unique; safer than raw pointers.
- `std::shared_ptr` — Reference counted; verify no circular ownership.
- `std::weak_ptr` — Non-owning; for breaking cycles.
- Raw pointers in modern C++ — Often a finding worth flagging unless documented as non-owning.

### Move Semantics

- Moved-from object is in valid-but-unspecified state; using its value is wrong.

### Templates

- Generally type-safe; type confusion possible with reinterpret_cast / bit_cast in template instantiations.

## Concurrency

### Data Races

- C++11+ has memory model; data race on non-atomic is UB.
- `std::atomic<T>` for shared mutable state; or mutex.
- TOCTOU on shared state — same as other languages.

### `pthread`s

- `pthread_mutex_lock` without `unlock` in error path — Deadlock.
- Signal-handler safety — `printf` is not async-signal-safe.

## Code Execution Sinks

### `system`, `popen`, `execlp`

- `system("cmd " + userInput)` — Shell-interpreted command injection.
- `execlp("program", "arg1", userArg, NULL)` — Argv form; safer.
- `execve` — Direct exec; argv form.

### `dlopen`, `LoadLibrary`

- Dynamic library loading with user-controlled path — RCE.
- `LD_PRELOAD` / `LD_LIBRARY_PATH` environment manipulation pre-launch.

### `fork` + `exec` Race

- Window between fork and exec where child still has parent's privileges/file descriptors — TOCTOU surface.

## Cryptography

### Common C Crypto Libraries

- **OpenSSL** — Vast surface; verify version not vulnerable.
- **mbedTLS** — Smaller; same considerations.
- **libsodium** — High-level safe primitives; preferred for new code.
- **BoringSSL** — Google's OpenSSL fork; used by Chromium and gRPC.
- **GnuTLS** — Less common; verify configuration.

### Patterns

- `RAND_bytes` (OpenSSL) — CSPRNG; verify return value (returns 0 on failure).
- `rand()` from `<stdlib.h>` — NOT cryptographic.
- Constant-time comparison: `CRYPTO_memcmp` (OpenSSL), `sodium_memcmp`, `mbedtls_ct_memcmp`. `memcmp` is NOT constant-time.

### Padding Oracle

- CBC-mode decryption with separate padding-error vs. MAC-error responses → Padding oracle. Use AEAD modes (GCM) or encrypt-then-MAC.

### Custom Crypto

- Hand-rolled crypto in C/C++ is a finding by default. Even when "looks correct," timing side channels and edge cases require expert review.

## Network / Socket Code

### `recv` / `read` Loops

- Reading exactly `n` bytes typically requires loop; partial reads.
- Buffer size validation before read.

### Format Parsing

- Length-prefixed protocol fields where length is user-controlled — Always validate before allocating / using.
- Variable-length fields without length validation — UAF / OOB / OOM.

### TLS

- `SSL_VERIFY_PEER` setting — Default off in older OpenSSL; verify enabled.
- Hostname verification — Often separate from cert verification; verify both.
- Cipher suite selection — Avoid old / weak suites.

## File Handling

- `fopen(path, mode)` with user-controlled path — Path traversal.
- `realpath(path, NULL)` — Canonicalize; verify result starts with intended root.
- TOCTOU between `stat` and `open` — Use `fstat` after open, or `O_NOFOLLOW`, or `openat` with relative path.
- Symlink-following — Default for most ops; risk in privileged contexts.
- Race between filesystem operations — Same TOCTOU risks across many calls.
- Temp files: `mktemp` (race), `tmpfile` (deletes immediately), `mkstemp` (atomic create-and-open) — Use `mkstemp`.

## SQL / DB (Native Clients)

- libpq, MySQL C API, SQLite — Verify parameterized query usage; concat into raw is injection.
- Same rules as higher-level languages.

## Common Library Footguns

### `libcurl`

- `CURLOPT_SSL_VERIFYPEER = 0` — TLS bypass.
- `CURLOPT_FOLLOWLOCATION = 1` without target validation — SSRF chain.
- `CURLOPT_PROTOCOLS` — Restrict to HTTP/HTTPS to block file://, gopher://, etc.

### `libxml2`

- `XML_PARSE_NOENT` flag — Enables entity expansion; XXE.
- `xmlReadFile` with permissive options.

### `zlib`, `bzip2`, `lzma`

- Decompression bombs — Verify size limits.

### `OpenSSL`

- `EVP_*` API — Modern; preferred over low-level.
- `SSL_CTX_set_verify` — Verify configured.
- Random initialization: `RAND_load_file`, `RAND_seed`.

### `printf`-Like in Logging Libraries

- `syslog(priority, "...%s...", userInput)` — Format string injection if userInput as format.

### Embedded HTTP Servers (Mongoose, libmicrohttpd)

- Verify CVE-tracked version.
- Default configurations may be permissive.

## Tooling Recommendations

For C/C++ codebases, manual audit alone is insufficient. Recommend:

- **Compiler flags** — `-Wall -Wextra -Wpedantic -Wformat=2 -Wformat-security -Wstack-protector -Wstrict-overflow -fstack-protector-strong -D_FORTIFY_SOURCE=2`.
- **Sanitizers** — AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), ThreadSanitizer (TSan), MemorySanitizer (MSan). Run tests under sanitizers.
- **Static analyzers** — clang-tidy, Cppcheck, Coverity, PVS-Studio, Semmle/CodeQL, Infer.
- **Fuzzing** — libFuzzer, AFL, OSS-Fuzz; for parsers, network protocols, file format handlers.
- **CFI / CET** — Control-Flow Integrity / Control-flow Enforcement Technology — runtime defenses against ROP-style exploits.
- **PIE / ASLR / NX / Stack canaries** — Compiler/linker flags; verify binaries shipped with these.

Even with tooling, audit findings exist. Some patterns are systemic and worth flagging in the audit:

- Codebase-wide use of `strcpy`, `strcat`, `sprintf` instead of bounded variants.
- Hand-rolled string handling without RAII / safe wrappers.
- Reliance on raw pointers for ownership in C++.
- Use of unsafe-by-default crypto primitives (CBC without HMAC, custom MAC, weak randomness).
- Network parsers that build up state before length validation.
- Lack of compiler/linker hardening flags.

## Common Findings Patterns

- `strcpy(dest, src)` without length check — Buffer overflow.
- `system("cmd " + userInput)` — Command injection.
- `printf(userInput)` — Format string vulnerability.
- `malloc(user_n * sizeof(T))` — Multiplication overflow.
- Custom `memcmp`-style comparison for MAC — Timing leak.
- `XML_PARSE_NOENT` enabled — XXE.
- `CURLOPT_SSL_VERIFYPEER = 0`.
- `rand()` for cryptographic randomness.
- `MD5` / `SHA1` for security purposes.
- `mktemp` (race-vulnerable temp file).
- Missing compiler hardening flags (`-fstack-protector`, `-D_FORTIFY_SOURCE=2`, `-fPIE`).
- `unsafe`-by-design APIs used (`gets`, `scanf("%s")` without width).

## Recommendation Patterns

- Use bounded string functions (`strncpy` with explicit null, `strlcpy` if available, `snprintf`).
- Migrate to C++ string types (`std::string`) and containers; eliminate raw allocation.
- Use RAII for resource management; smart pointers for ownership.
- Use libsodium for new crypto code; verify high-level APIs.
- Use parameterized queries via library APIs.
- Compile with hardening flags; run tests under sanitizers; integrate fuzzing for parsers.
- Use safe wrappers from `Abseil`, `Folly`, `Boost.SafeInt` for arithmetic safety.
- For new development in security-critical contexts, consider Rust as alternative; for existing C/C++, isolate untrusted-input parsing in sandboxed processes or memory-safe wrappers.
- Pin / verify all third-party C/C++ libraries; track CVEs aggressively.
- Document `unsafe`-equivalent patterns explicitly so reviewers can focus.
