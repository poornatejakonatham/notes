# The Go toolchain

The `go` executable is the main application of the Go toolchain. You can pass a command to `go`, and it will take the appropriate action. The toolchain has tools to run, compile, format source code, download dependencies, and more. Let's look at the full list, which is obtained as an output from the `go help` command or just `go` by itself:

- `build`: This compiles packages and dependencies
- `clean`: This removes object files
- `doc`: This shows documentation for a package or symbol
- `env`: This prints Go environment information
- `generate`: This is the code generator
- `fix`: This upgrades Go code when a new version is released
- `fmt`: This runs `gofmt` on package sources
- `get`: This downloads and installs packages and dependencies
- `help`: This provides more help on a specific topic
- `install`: This compiles and installs packages and dependencies
- `list`: This lists packages
- `run`: This compiles and runs Go programs
- `test`: This runs unit tests and benchmarks
- `vet`: This examines source code for bugs
- `version`: This shows the Go version

More information about these commands is available at [https://golang.org/cmd/](https://golang.org/cmd/).