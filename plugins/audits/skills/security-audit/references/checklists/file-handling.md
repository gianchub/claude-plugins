# File Handling

## Scope

Path traversal, arbitrary file read/write, file uploads (validation, storage, serving), archive extraction (zip slip), polyglot files, image library vulnerabilities, and temporary file handling. Anywhere the application reads, writes, or processes files where any input (path, content, name, type) is influenced by untrusted data.

## Path Traversal

User input flows into a file path, allowing access to files outside the intended directory.

### Patterns

- **`open(userInput)` / `File.read(userInput)`** — User input directly as path. Any `../` sequences escape the working directory.
- **`open(BASE + userInput)`** — Concatenation with a base path; still allows `../../etc/passwd` to escape.
- **`os.path.join(base, userInput)` / `path.join(base, userInput)`** — Does not block `../`; on most platforms, `path.join('/safe', '../etc/passwd')` returns `/etc/passwd`. Verify post-join with canonicalization-and-prefix-check.
- **`Paths.get(base, userInput)`** (Java) — Same risk; use `resolve()` and check `startsWith()` after `normalize()`.
- **Encoded traversal** — `%2e%2e/`, `%252e%252e/`, `..%c0%af` (overlong UTF-8). Decoding may happen at multiple layers; validate after final decoding, or use canonicalization.
- **Null-byte injection** — `userInput + ".txt"` with `userInput = "../etc/passwd\x00"` strips suffix in C-based libs. Reject `\x00`.
- **Backslash on Windows** — `..\\..\\windows\\win.ini`; some libs treat `/` as path separator and ignore `\`.
- **UNC paths on Windows** — `\\\\attacker.example.com\share\...` — file APIs may follow.

### Defenses

- **Allowlist filenames** — Best defense when the file set is known: `if filename in {"a", "b", "c"}: open(path / filename)`.
- **Canonicalize then check prefix** — Resolve to absolute path with `..` collapsed (`os.path.realpath`, `Paths.get(...).toRealPath()`, `path.resolve()`, `Path.GetFullPath()`), then verify the result is within the intended directory:
  ```python
  base = Path("/srv/uploads").resolve()
  candidate = (base / userInput).resolve()
  if base not in candidate.parents:
      raise PermissionError
  ```
- **Sanity-check filenames** (not a substitute for canonicalization) — Reject names containing `/`, `\`, or `..` segments before any path operation. Use this only as a fast-fail layer; canonicalization-and-prefix-check is still required.
- **chroot / containers** — OS-level scoping; defense-in-depth.

### Common Sinks

- File serving endpoints (`/static/${name}`).
- Download endpoints (`?file=...`).
- Template / view rendering by name (`render_template(userTemplate)`).
- Include / require with user input (`include $name . ".php"`).
- Log file path / output file path configurable via API.

## File Uploads

### Filename Handling

- **Use server-generated names** — Don't preserve user-supplied filenames; generate UUIDs or content-hash-based names. Eliminates filename-injection class entirely.
- **If preserving filenames** — Sanitize: strip path separators, control chars, leading dots, reserved Windows names (`CON`, `PRN`, `AUX`, `NUL`, `COM1-9`, `LPT1-9`), trailing dots and spaces.
- **Length limits** — Enforce max length on filename; some filesystems silently truncate.
- **Unicode normalization** — Normalize to NFC; reject names that change after normalization.

### Content-Type / Type Validation

- **Don't trust client `Content-Type`** — Attacker-controllable. Validate by sniffing magic bytes server-side.
- **Magic-byte detection** — Use a library (`python-magic`, `mimetype-io`, `apache-tika`); not just file extension.
- **Allowlist content types** — Reject anything not on the allowlist explicitly.
- **Polyglot files** — Files valid in multiple formats (JPEG that's also a valid PHP file, PNG that's also a valid SVG). Risk depends on how the application later interprets the content.
- **Image-as-script** — Uploading "image.php.jpg" or relying on Apache mod_mime parsing the inner extension.
- **SVG with script** — SVG is XML and runs JS in browser context. If served from app origin: stored XSS. Sanitize SVG server-side or serve with `Content-Disposition: attachment` and a non-app domain (sandbox domain pattern).
- **HTML uploads** — Same problem as SVG; never serve user HTML from the app domain without strict sandboxing.

### File Size and Quantity

- **Per-request size limits** — Enforced at framework or web server level (`client_max_body_size` in nginx, `LimitRequestBody` in Apache, `app.use(express.json({ limit: '1mb' }))`).
- **Per-file size limits** — Enforced before reading the entire upload into memory; streaming is essential for large files.
- **Per-user / per-tenant quota** — Storage quota enforced; abuse vector otherwise.
- **Total request count limits** — Rate limiting on the upload endpoint.

### Storage

- **Don't store under web-document root** — Uploads accessible by URL must go through application code, not direct file serving (or be on a separate sandbox domain).
- **Permissions** — Stored files not executable; uploaded files not readable by unrelated users on a shared host.
- **Cloud object storage** — When using S3/GCS/Azure Blob: signed URLs for download, ACL-private by default, bucket policy denying public read.
- **Antivirus / content scanning** — For applications accepting files for distribution to other users (file sharing, support attachments). Document if used; flag absence in such contexts.

### Serving / Download

- **`Content-Disposition: attachment`** — Forces download instead of inline render; recommended for arbitrary user content.
- **`X-Content-Type-Options: nosniff`** — Prevents browser MIME sniffing; combined with correct `Content-Type` header.
- **CORS / origin** — Files served from a sandbox origin separate from the app origin; cookies / app context not exposed.
- **Authorization on download** — Even for "user-uploaded" content, verify the requester is permitted to download.
- **Preview generation** — When the application renders previews (PDF thumbnails, image resizing), the rendering path is its own attack surface (image library CVEs).

## Archive Extraction (Zip Slip and friends)

### Zip Slip

- Archive entry filename containing `../` causing extraction to write outside the intended directory.
- **Per-language risk**:
  - Java: `ZipInputStream` requires manual canonicalization; many implementations are vulnerable.
  - Python: `tarfile.extractall` and `zipfile.extractall` are vulnerable to absolute paths and `..`. Use a member-by-member loop with canonicalization. Python 3.12+ adds `filter='data'` for safer tar extraction.
  - Node: `unzipper`, `adm-zip`, `tar` historically had Zip Slip issues; verify version.
  - Go: `archive/zip`, `archive/tar` — manual canonicalization required.

### Defense

- For each entry: resolve full path, verify `startsWith(extractRoot)`. Reject entry otherwise.
- Reject absolute paths in entries.
- Reject symlinks in archive entries on extract (or follow with strict canonicalization).
- Limit total uncompressed size (anti-zip-bomb).
- Limit entry count (anti-many-small-files DoS).

### Zip Bomb

- Highly compressed archives (1KB → 1GB ratio) exhaust memory or disk.
- Mitigate by streaming with size limit; abort when uncompressed size exceeds threshold.
- Recursive zips (zip in zip in zip): stop at depth 1 unless intentional.

### Other Archive Types

- TAR with hardlinks/symlinks — same Zip Slip class via link targets. Reject symlinks/hardlinks during extract or canonicalize against root.
- 7z, RAR, ACE — additional formats; treat with same caution.

## Image / Document Libraries

Third-party libraries that process user-uploaded files have repeated CVE history. Audit which libraries are present and their versions.

### ImageMagick

- **CVE-2016-3714 ("ImageTragick")** — Command injection via crafted SVG/MVG.
- Modern versions safer with `policy.xml` restricting coders. Verify policy is restrictive (disable HTTPS, MVG, MSL, FTP, EPHEMERAL, etc.).

### libvips, Sharp (Node binding)

- Generally safer than ImageMagick; still update for CVEs.

### Pillow / PIL (Python)

- Multiple historical CVEs in image format parsers.
- `Image.open(...)` can trigger external process invocation for some formats; verify config.

### PDF Libraries

- `pypdf2`, `pdf-lib`, `pdf2htmlEX` — historic CVEs.
- PDF JavaScript / form-action features are attack surface.
- Server-side PDF rendering (wkhtmltopdf, WeasyPrint) is also SSRF surface (covered in `ssrf-redirect-url.md`).

### Office Document Libraries

- `python-docx`, `openpyxl`, `apache-poi`: macro and XML-formula injection risks.
- Excel CSV / XLSX formulas (`=cmd|'/c calc'!A1`) execute when opened — CSV injection (formula injection); covered below.

### Video / Media

- ffmpeg has rich CVE history; isolate and version-track.

## CSV / Formula Injection

When the application generates CSV files containing user-provided data, content beginning with `=`, `+`, `-`, `@`, `\t`, or `\r` is interpreted as a formula by Excel and Google Sheets when the file is opened.

- Prefix risky leading chars with a single quote (`'`) or tab. Escape these characters.
- Document the risk if the CSV is intended for technical consumers — but err on side of escaping.

## Temporary Files

- **`mktemp`-style insecure naming** — Predictable temp filenames; symlink attacks. Use `mkstemp` (C), `tempfile.NamedTemporaryFile` (Python), `Files.createTempFile` (Java), `os.MkdirTemp` (Go).
- **`/tmp` permissions** — World-readable on multi-user hosts; sensitive data must be in app-specific directory.
- **Cleanup** — Temp files removed in error paths; `with` / `try-finally` / `defer`.
- **Concurrent use** — Same temp file used by multiple processes is a TOCTOU vector.

## File Permissions on Output

- Files created by the application have restrictive default permissions (umask 0027 or 0077 typical for sensitive content).
- Public-readable output (e.g., generated reports for download) — verify no sensitive content.

## Symbolic Links

- Application following user-controlled symlinks may read outside intended scope.
- `os.lstat` to check before opening; or open with `O_NOFOLLOW` flag.
- Symlinks within uploaded archives — covered above.

## Recommendation Patterns

- Always canonicalize paths and check prefix; never rely on string-level traversal blocks.
- Generate server-side filenames; preserve user's name only as metadata.
- Validate file content by magic bytes, not by extension or client-supplied Content-Type.
- For archive extraction, validate every entry's resolved path.
- Limit upload size, rate, and storage quota.
- Serve user-uploaded content from a separate origin or via attachment-only download, with explicit Content-Type and `nosniff`.
- Track image/PDF library versions; CVEs are frequent.
- Run media-processing pipelines in sandboxed processes (containers, ulimit, seccomp).
