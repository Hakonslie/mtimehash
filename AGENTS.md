# AGENTS.md

Guide for AI agents working on the mtimehash codebase.

## Project Overview

**mtimehash** is a Go CLI tool that modifies file modification times based on the hash of file content. This makes mtimes deterministic, which improves `go test` caching in CI environments where git doesn't preserve timestamps.

## Commands

### Build
```bash
# Build the CLI binary
go build -o mtimehash ./cmd/mtimehash

# Install globally
go install github.com/slsyy/mtimehash/cmd/mtimehash@latest
```

### Test
```bash
# Run all tests with race detection
go test -v -race ./...

# Run tests for specific package
go test -v ./mtimehash
go test -v ./cmd/mtimehash
```

### Lint
```bash
# Run golangci-lint (version 1.63)
golangci-lint run

# Configuration in .golangci.yml - enables gofumpt formatter only
```

### Using the CLI
```bash
# Process files (pass via stdin)
find . -type f | ./mtimehash

# With options
find . -type f -size -10000k ! -path ./.git/** | ./mtimehash --verbose --max-unix-time=1704067200

# Enable CPU profiling
find . -type f | ./mtimehash --cpu-profile-path=profile.out
```

## Code Organization

```
/
├── mtimehash.go              # Core library: Process() function
├── mtimehash_test.go         # Unit tests for core library
├── cmd/mtimehash/
│   ├── main.go               # CLI entry point with urfave/cli
│   ├── main_test.go          # Integration tests using go-cmdtest
│   └── testdata/
│       └── main.ct           # Command test specifications
├── go.mod                    # Go 1.23.4, module: github.com/slsyy/mtimehash
├── .golangci.yml             # Lint config (gofumpt only)
└── .github/workflows/test.yml # CI: tests on ubuntu/macos, golangci-lint on ubuntu
```

## Key Libraries & Dependencies

- **CLI**: `github.com/urfave/cli/v2` - Command-line interface
- **Concurrency**: `github.com/sourcegraph/conc` - Structured goroutine pooling
- **Testing**: 
  - `github.com/stretchr/testify` - Assertions and helpers
  - `github.com/google/go-cmdtest` - CLI integration testing
- **Performance**: `go.uber.org/automaxprocs` - Automatic GOMAXPROCS tuning
- **Crypto**: Standard library `crypto/sha256` - File hashing

## Architecture

### Library (`mtimehash.go`)
- **Main API**: `Process(input iter.Seq[string], maxUnixTime int64) error`
- **Concurrency**: Uses `conc/pool` with `runtime.GOMAXPROCS(0)` goroutines
- **Hashing**: SHA256 of file content → first 8 bytes → uint64
- **Timestamp**: `hash % maxUnixTime` converted to `time.Time`
- **Logging**: Uses `slog` for structured logging with Debug/Error levels

### CLI (`cmd/mtimehash/main.go`)
- **Input**: Reads file paths from stdin (newline-separated)
- **Flags**:
  - `--max-unix-time` (default: 1704067200 - beginning of 2024)
  - `--verbose, -v` - Enable debug logging
  - `--cpu-profile-path` - Write CPU profile to file
- **Logging**: Text format to stderr, configurable level
- **Signals**: Handles profiling start/stop, ignores HUP/TERM (process exits naturally)

## Testing Strategy

### Unit Tests (`mtimehash_test.go`)
- Tests `Process()` with various scenarios
- Validates deterministic mtimes from content hashes
- Tests error handling for non-existent files, directories, permission errors
- Uses `t.TempDir()` for isolated file fixtures

### Integration Tests (`cmd/mtimehash/main_test.go`)
- Uses `github.com/google/go-cmdtest`
- Tests via `testdata/main.ct` file with command patterns
- Runs CLI in-process for reliable testing
- Defines custom test commands (e.g., `fileNotEmpty`)
- `silentLogs` flag suppresses logging during tests

### Test Patterns
- **Assertion style**: Prefer `testify/require` for setup, `testify/assert` for checks
- **File fixtures**: Always use `t.TempDir()` for temporary files
- **Error cases**: Test that valid files still get processed when some files error

## Code Style & Conventions

### Formatting
- Uses `gofumpt` (stricter than gofmt) via golangci-lint
- No manual formatting rules needed - rely on `gofumpt`

### Error Handling
- Wrap errors with `fmt.Errorf`: `%w` verb
- Log errors before returning (unless in CLI main)
- Always handle `defer` cleanup properly

### Logging
- Use `slog.Default()` for structured logging
- Debug level: Operational details
- Error level: Failures that need attention
- Use key-value pairs: `slog.Default().Error("message", "key", value)`

### Concurrency
- Always use structured concurrency via `conc` package
- `p := pool.New().WithErrors().WithMaxGoroutines(n)`
- Return `p.Wait()` to bubble up errors from goroutines

## Gotchas & Important Notes

1. **Non-regular files**: The library explicitly rejects non-regular files (directories, symlinks, etc.) with helpful error messages
2. **Fails partially**: If some files error, Process() returns an error but still processes all files it can
3. **Race detector**: Always run with `-race` in CI and during dev
4. **Build artifact**: Binary is named `mtimehash` and gitignored (do not commit)
5. **Go version**: Requires Go 1.23.4 (mod file)
6. **CI platforms**: Tests run on both Ubuntu and macOS in CI

## Contributing

- Ensure tests pass: `go test -v -race ./...`
- Lint passes: `golangci-lint run`
- Integration tests: Verify by running actual CLI with find + mtimehash
- No breaking changes to public API without good reason (it's a library too)

## License

MIT License - Krystian Chmura (2025)
