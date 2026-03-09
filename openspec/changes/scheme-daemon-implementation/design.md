# Design: Scheme Daemon Implementation

## Architecture

Both daemons follow the exact pattern established by `virtio-fsd`:

```
Socket::create()
  → register_sync_scheme(&socket, "store", &mut handler)
  → loop { socket.next_request() → call.handle_sync(&mut handler) → socket.write_response() }
```

The `SchemeSync` trait dispatches to methods: `openat`, `read`, `write`, `fstat`,
`fsize`, `fpath`, `getdents`, `on_close`. Each method delegates to the existing
core modules.

## Dependencies

Add to `Cargo.toml` behind `#[cfg(target_os = "redox")]`:
- `redox_scheme` — Socket, SchemeSync trait, OpenResult, CallerCtx
- `syscall` — Stat, DirentBuf, error codes, flags, schemev2::NewFdFlags

These are already used by `virtio-fsd` and available in the Redox ecosystem.
Since snix is built via `mkUserspace.mkBinary`, we need to ensure these crates
are available in the cargo vendor directory.

## stored SchemeSync

```
openat(path) → resolve::parse_scheme_path(path)
             → lazy::ensure_extracted(if needed)
             → handles.open_file() or handles.open_dir()
             → OpenResult::ThisScheme { number: id }

read(id, buf, offset) → handles.read(id, buf, offset)

fstat(id, stat) → stat from underlying fs::File metadata

getdents(id, buf) → handles.list_dir(id) for dirs
                   → resolve::list_store_paths(db) for root

fpath(id, buf) → write "store:{scheme_path}" to buf

fsize(id) → handles.file_size(id)

on_close(id) → handles.close(id)
```

## profiled SchemeSync

```
openat(path) → parse profile name + subpath
             → if ".control": open control handle
             → else: mapping.resolve_path(subpath) → open underlying file
             → OpenResult::ThisScheme { number: id }

read(id, buf, offset) → file handle: read from underlying file
                       → control handle: return EINVAL (write-only)

write(id, buf) → control handle: process_control(json)
              → file handle: return EACCES (read-only profiles)

getdents(id, buf) → mapping.list_union(subpath) for profile dirs
                   → profiles.list_profiles() for root

fpath(id, buf) → "profile:{scheme_path}"

on_close(id) → close handle, drop control buffer
```

## Handle Types

### stored
- `FileHandle`: wraps `fs::File` opened from `/nix/store/...`
- `DirHandle`: wraps directory path with cached entries
- Both already exist in `stored/handles.rs`

### profiled
- `FileHandle`: wraps `fs::File` resolved through profile mapping
- `DirHandle`: wraps a subpath for union listing
- `ControlHandle`: accumulates write data for JSON command processing
- Need a new handle table in profiled (simpler than stored — reuse the pattern)

## Testing

- Unit tests remain on Linux (mock filesystems, no scheme registration)
- VM functional tests boot with the daemons in init.d, then exercise scheme paths
- Test stored: `cat store:hash-pkg/bin/rg` via bash, verify content
- Test profiled: add package via .control, read through profile: scheme
- Test fallback: daemons not running → direct filesystem works
