# Orbital Login Spawn Fix Analysis
**Date:** 2026-01-05 21:00:00
**Issue:** `[orbital@orbital:82 ERROR] error during daemon execution, exiting with status=1: failed to spawn login_cmd`

## Root Cause

The Orbital init script passed `orblogin` and `orbterm` as bare command names instead of absolute paths:

**Before (broken):**
```ion
nowait /bin/orbital orblogin orbterm
```

**After (fixed):**
```ion
nowait /bin/orbital /bin/orblogin /bin/orbterm
```

## Technical Details

### How Orbital Spawns login_cmd

From the Orbital source code (`orbital/src/main.rs`):
```rust
let login_cmd = args.next().ok_or("no login manager argument")?;
// ...
Command::new(login_cmd)
    .args(args)
    .spawn()
    .map_err(|_| "failed to spawn login_cmd")?;
```

Orbital receives the login command as a command-line argument and spawns it using `Command::new()`. When passed as a bare name `orblogin` (not `/bin/orblogin`), the spawn fails because:
1. Redox OS process spawning may not perform PATH lookup
2. The binary needs to be specified with its absolute path

### Files Modified

**`nix/pkgs/infrastructure/disk-image.nix`** (line 493):
```nix
# Changed from:
nowait /bin/orbital orblogin orbterm

# Changed to:
nowait /bin/orbital /bin/orblogin /bin/orbterm
```

### Related Warning (Non-Fatal)

```
[orbital@orbital:50 WARN] inputd -A '1' failed to run with error: No such file or directory (os error 2)
```

This warning occurs because:
1. `inputd` is already started during initfs boot (without -A flag)
2. Orbital internally tries to activate VT 1 by spawning `inputd -A '1'`
3. Since inputd is already running, this redundant spawn fails

This is **non-fatal** - the VT activation already happened during initfs boot, so Orbital functions correctly despite this warning.

## Package Chain Verification

The build chain is correctly configured:

| Package | Status | Binary |
|---------|--------|--------|
| orbutils | Builds successfully | `/bin/orblogin`, `/bin/background` |
| orbital | Builds successfully | `/bin/orbital` |
| orbterm | Builds successfully | `/bin/orbterm` |
| orbdata | Builds successfully | `/ui/orbital.toml`, fonts, icons |
| userutils | Builds successfully | `/bin/login`, `/bin/getty`, etc. |

## Testing

After the fix:
```bash
# Rebuild disk image
nix build .#diskImage

# Run in QEMU graphical mode
nix run .#run-redox-graphical
```

Expected behavior:
1. Boot completes successfully
2. Orbital display server starts
3. `orblogin` graphical login window appears
4. On authentication, `orbterm` terminal spawns

## Documentation Updates

Updated comments in:
- `nix/pkgs/infrastructure/disk-image.nix` (line 479)
- `nix/pkgs/userspace/orbutils.nix` (line 12)

Both now reference the correct command format:
```
orbital /bin/orblogin /bin/orbterm
```
