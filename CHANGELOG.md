# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Detailed media metadata extraction documentation with two strategies:
  - Pure Go libraries (imagemeta, dhowden/tag, go-mp4, pdfcpu)
  - FFprobe integration for video
- Media metadata dependencies in go.mod specification
- `media.go` in harvest package structure
- Comprehensive README.md with full architecture documentation
- CLI interface design with all commands and options
- Database schema design for SQLite + sqlite-vec
- Implementation details including directory structure and key interfaces
- Configuration documentation via environment variables
- Tech stack and dependency specifications
- Development roadmap with phased milestones

### Documentation
- Defined search command: `sem <query>` with flags `-n`, `-s`, `-c`, `-j`, `-t`, `-e`
- Defined index command: `sem index [paths...]` with flags `-f`, `-e`, `-x`, `-d`
- Defined status command: `sem status` with flags `-j`, `-s`
- Defined global flags: `-v`, `-h`, `-V`
- Documented environment variables: `SEM_EMBEDDER`, `SEM_DB`, `SEM_CONTEXT`, `SEM_RESULTS`, `SEM_VERBOSE`
- Specified pipeline architecture: Harvest → Embed → Store → Query
- Designed concurrency model with worker pool
