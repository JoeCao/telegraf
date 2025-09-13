# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Telegraf is a plugin-driven server agent for collecting, processing, aggregating, and writing metrics, logs, and other arbitrary data. It's written in Go and uses TOML configuration files.

## Development Commands

### Building
- `make all` - Download dependencies and compile telegraf binary
- `make build` - Compile telegraf binary only
- `make telegraf` - Alias for build

### Testing
- `make test` - Run short unit tests with race detector
- `make test-integration` - Run integration tests only
- `make test-all` - Run full test suite with format checks and vet

### Linting and Code Quality
- `make lint` - Run golangci-lint and markdownlint
- `make lint-install` - Install linters (golangci-lint v2.4.0, markdownlint-cli)
- `make fmt` - Format Go source files
- `make fmtcheck` - Check if code is formatted correctly
- `make vet` - Run go vet
- `make check` - Run fmtcheck and vet together
- `make tidy` - Verify and tidy Go modules

### Documentation
- `make docs` - Generate and embed documentation into plugin READMEs
- `make config` - Generate default telegraf.conf configuration file

### Pre-PR Checklist
Before opening a pull request, run:
```bash
make lint
make check
make check-deps
make test
make docs
```

## Architecture Overview

### Plugin System
Telegraf uses a plugin-based architecture with five main plugin types:
- **Input plugins** (`plugins/inputs/`) - Collect metrics from various sources
- **Output plugins** (`plugins/outputs/`) - Write metrics to various destinations
- **Processor plugins** (`plugins/processors/`) - Transform metrics between input and output
- **Aggregator plugins** (`plugins/aggregators/`) - Create aggregate metrics
- **Secret store plugins** (`plugins/secretstores/`) - Retrieve secrets for configuration

### Core Components
- **Agent** (`agent/`) - Main orchestration and plugin management
- **Config** (`config/`) - Configuration parsing and validation
- **Internal** (`internal/`) - Shared internal libraries and utilities
- **Models** (`models/`) - Core data structures for metrics
- **Serializers/Parsers** (`plugins/serializers/`, `plugins/parsers/`) - Data format handling

### Configuration
- Uses TOML format exclusively
- Generate config: `telegraf config > telegraf.conf`
- Filter plugins: `telegraf config --input-filter cpu:mem --output-filter influxdb`

## Development Guidelines

### Go Module Management
- Project uses Go 1.25.0
- Dependencies managed via `go.mod`/`go.sum`
- Run `make tidy` to verify and clean modules

### Plugin Development
- Each plugin type has specific development docs in `/docs/`
- Follow existing plugin patterns in respective directories
- All plugins must include proper unit tests and documentation
- External plugins can be run via execd plugins without code changes

### Code Standards
- Follow golangci-lint configuration in `.golangci.yml`
- Use conventional commit format for PR titles
- AI-generated code contributions are not accepted

### Testing
- Short tests for quick feedback during development
- Integration tests for end-to-end functionality
- Race detector enabled (64-bit architectures only)
- Test containers available for integration testing

## Important Files
- `Makefile` - Build system and development commands
- `.golangci.yml` - Linter configuration
- `go.mod` - Go module dependencies
- `CONTRIBUTING.md` - Contribution guidelines and workflows
- `etc/telegraf.conf` - Generated default configuration