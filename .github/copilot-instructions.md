# SyncLink - AI Coding Agent Instructions

## Project Overview

SyncLink is a Windows-focused CLI tool (inspired by Scoop's `persist` feature) that helps manage configuration files by moving them to a centralized sync directory (Dropbox, OneDrive, etc.) and creating symbolic links or shortcuts at the original location. Written in Go with Cobra for CLI structure.

**Core Use Case**: Applications that store configs in non-standard locations (e.g., `%APPDATA%\Local\uv`) which sync tools don't natively handle. SyncLink automates move-and-link operations for seamless backup/sync across machines.

## Architecture

### Package Structure

- **`cmd/`**: Cobra command definitions (link, unlink, relink, list, config)
  - Each command is self-contained in its own file
  - `root.go` initializes config loading via `PersistentPreRunE`
- **`internal/config/`**: JSON-based configuration management
  - Singleton pattern with `sync.Once` for thread-safe loading
  - Config file stored alongside executable as `config.json`
  - Structure: `Settings` (global), `Links` map (tracked items)
- **`internal/link/`**: Core link/shortcut creation logic
  - `link.go`: Cross-platform link management (symlinks)
  - `shortcut_windows.go`: Windows-specific COM-based `.lnk` creation using `go-ole`
  - Delegate pattern for platform-specific implementations
- **`internal/util/`**: Shared utilities (path handling, file ops, colored console output)

### Data Flow

1. **Link Creation** (`synclink link <target>`):
   - Validate target exists → Move to sync dir (files go to `{sync_path}/files/{name}`, folders to `{sync_path}/{name}`)
   - Create symlink at original location → Update `config.json`
   - Optional: Create Start Menu shortcut via `--shortcut` flag
2. **Unlink** (`synclink unlink <name>`):
   - If symlink: Move data back to original path → Delete symlink
   - If shortcut: Remove `.lnk` file
   - Remove from config
3. **Relink** (`synclink relink <name|*>`):
   - Check if symlink/shortcut exists and is valid
   - Recreate if missing/broken using stored config metadata

### Configuration Storage

```json
{
  "settings": {"default_sync_path": ""},
  "links": {
    "linkName": {
      "shortcut": false,
      "original_path": "C:\\...",
      "synced_path": "D:\\Sync\\...",
      "created_at": "2025-..."
    }
  },
  "version": "1.0"
}
```

- Config lives in executable directory (portable design)
- Access via singleton: `config.GetConfig()` (thread-safe)
- Mutate with `config.ConfigMutex` locking

## Development Conventions

### Error Handling

- Always wrap errors with context: `fmt.Errorf("operation failed: %w", err)`
- Use `util.ErrorPrint()` / `util.WarningPrint()` for colored stderr output
- Rollback on failure (see `CreateSymbolicLink` - moves file back if symlink fails)

### Path Management

- **Always** use absolute paths internally via `util.GetAbsPath()`
- Windows-specific: Handle `filepath.Join()` for cross-disk moves
- File vs folder distinction: Files go to `{sync}/files/`, folders to `{sync}/{name}`

### Concurrency

- Config loading is thread-safe (singleton with `sync.Once`)
- Use `config.ConfigMutex` for write operations (already handled in `SaveConfig`)
- `relink *` and `unlink *` support batch operations with error aggregation

### COM/Windows Integration

- Windows shortcuts use OLE automation via `go-ole` library
- Always call `ole.CoInitializeEx()` and defer `ole.CoUninitialize()`
- Check for `S_FALSE` (already initialized) in COM init errors
- See `shortcut_windows.go:createShortcutWindows()` for pattern

## Build & Test

```powershell
# Build
go build -o synclink.exe

# Install dependencies
go mod download

# Test (no automated tests exist yet)
# Manual testing recommended: Create test directory structure and verify link/unlink cycles
```

## Key Design Decisions

1. **Config in executable directory**: Enables portable deployment (drag-drop executable + config)
2. **Separate files/ subdirectory**: Prevents name collisions between files and folders
3. **Wildcard support** (`*`): Allows batch operations like `synclink relink *` to fix all links
4. **Delegate pattern for shortcuts**: Abstracts Windows-specific COM logic from core link management
5. **No cross-platform shortcuts**: Only Windows `.lnk` via COM (symlinks work on Unix but untested)

## Common Tasks

### Adding a New Command

1. Create `cmd/newcmd.go` with `cobra.Command` struct
2. Add to `rootCmd` in `init()` function
3. Use `config.GetConfig()` for access to managed links
4. Follow error wrapping pattern and colored output for consistency

### Modifying Config Structure

1. Update structs in `internal/config/config.go`
2. Increment `CurrentVersion` constant
3. Add migration logic in `LoadConfig()` (see TODO at line 157)
4. Update `config.json` schema in workspace root and `build/` folder

### Working with Symlinks

- Use `os.Symlink(target, linkPath)` (works on Windows with dev mode or elevated privileges)
- Detect existing symlinks: `os.Lstat()` → check `info.Mode()&os.ModeSymlink`
- See `util.IsSymlink()` for helper function

## Known Limitations

- **Windows-only shortcuts**: COM automation requires Windows
- **Cross-disk moves**: Basic implementation (see TODO in util for progress bars)
- **No automated tests**: Rely on manual testing with real file systems
- **Version migration**: Placeholder exists but not implemented
- **Elevated privileges**: Symlinks may require developer mode or admin rights on Windows

## References

- Cobra docs: https://github.com/spf13/cobra
- go-ole: https://github.com/go-ole/go-ole
- Windows symlinks: Require SeCreateSymbolicLinkPrivilege or Developer Mode
