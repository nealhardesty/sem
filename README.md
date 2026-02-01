# sem

**Local Semantic Search via CLI**

Search code by **intent**, not keywords. `sem` provides fast, local vector indexing of your filesystem using Tree-sitter parsing and MiniLM embeddings.

```bash
sem "token refresh logic"           # Find code by meaning
sem -s ./internal "db connection"   # Scope to directory
sem "error handling" -j | gx "..."  # Pipe JSON to other tools
```

## Installation

```bash
go install github.com/nealhardesty/sem@latest
```

**Requires:** Go 1.21+, SQLite 3.x

## Usage

```
sem [options] <query>    Search (auto-indexes if needed)
sem -V                   Version
sem -h                   Help
```

### Flags

| Flag | Long | Default | Description |
|------|------|---------|-------------|
| `-n` | `--num` | `10` | Number of results |
| `-s` | `--scope` | all | Limit to directory path |
| `-c` | `--context` | `3` | Context lines around matches |
| `-j` | `--json` | `false` | JSON output |
| `-t` | `--threshold` | `0.5` | Similarity threshold (0.0-1.0) |
| `-f` | `--force` | `false` | Force re-index |
| `-x` | `--exclude` | gitignore | Additional exclude patterns |
| `-v` | `--verbose` | `false` | Verbose output |

### Output

```
[1] ./internal/auth/token.go:45-67 (score: 0.89)
    func RefreshToken(ctx context.Context, token string) (*Token, error) {
        ...
    }
```

## Architecture

```
HARVEST ──▶ EMBED ──▶ STORE ──▶ QUERY

Tree-sitter   MiniLM     SQLite     Vector
File Walker   ONNX       sqlite-vec Similarity
```

### Harvester

Extracts semantic chunks from files:

- **Code** (Tree-sitter): functions, methods, structs, interfaces, types
- **Text**: 512-byte overlapping chunks
- **Config** (YAML/JSON/TOML): structural parsing
- **Media**: rich metadata extraction (see below)

**Languages:** Go, Python, JS/TS, Rust, Java, C/C++, Ruby, PHP, Swift, Kotlin

### Media Metadata Extraction

Multiple strategies available, from pure Go (zero dependencies) to external tools (maximum coverage):

#### Strategy 1: Pure Go Libraries (Recommended)

No external dependencies, cross-platform, fast.

| Media Type | Library | Features |
|------------|---------|----------|
| **Images** | `github.com/dsoprea/go-exif/v3` | Full EXIF/IFD traversal, GPS, timestamps |
| | `github.com/evanoberholster/imagemeta` | EXIF + XMP + IPTC unified API |
| | `github.com/rwcarlsen/goexif` | Simple EXIF (JPEG/TIFF only) |
| **Audio** | `github.com/dhowden/tag` | ID3v1/v2, FLAC, OGG Vorbis, MP4/M4A |
| | `github.com/bogem/id3v2` | Full ID3v2.3/v2.4 read/write |
| **Video** | `github.com/abema/go-mp4` | MP4/M4V box-level parsing |
| | `github.com/alfg/mp4` | MP4 container metadata |
| **PDF** | `github.com/pdfcpu/pdfcpu` | PDF metadata, page count, encryption |

**Extracted fields (images):** camera make/model, GPS coordinates, timestamps, dimensions, orientation, lens info, exposure settings, software

**Extracted fields (audio):** title, artist, album, year, track, genre, duration, bitrate, sample rate, album art

**Extracted fields (video):** duration, resolution, codec, fps, bitrate, creation time

#### Strategy 2: FFprobe (Maximum Video Coverage)

Best for video files. Requires `ffmpeg` installed.

```go
// Via os/exec - returns JSON with all stream/format metadata
cmd := exec.Command("ffprobe", "-v", "quiet", "-print_format", "json", 
    "-show_format", "-show_streams", path)
```

**Pros:** Supports virtually all video/audio formats (MKV, AVI, WebM, etc.)
**Cons:** External dependency, slower startup

#### Recommended Approach

```
┌─────────────────────────────────────────────────────────┐
│                    File Type Router                      │
├──────────────┬──────────────┬──────────────┬────────────┤
│    Images    │    Audio     │    Video     │    PDF     │
│  imagemeta   │  dhowden/tag │  ffprobe OR  │  pdfcpu    │
│  (pure Go)   │  (pure Go)   │  go-mp4      │  (pure Go) │
└──────────────┴──────────────┴──────────────┴────────────┘
```

- Use pure Go for images/audio/PDF (fast, no deps)
- Use FFprobe for video if available, fall back to go-mp4 for MP4

#### Metadata → Embeddings

Media metadata is converted to searchable text:

```
Photo: Canon EOS R5, 24mm f/2.8, ISO 400, 2024-03-15 14:32:00
Location: 37.7749°N 122.4194°W (San Francisco, CA)
---
Audio: "Hotel California" by Eagles, Album: Hotel California (1977)
Genre: Rock, Duration: 6:30, 320kbps MP3
```

This text is then embedded alongside filename for semantic search.

### Embedding Engines

| Engine | ID | Dims | Notes |
|--------|-----|------|-------|
| MiniLM | `minilm` | 384 | Default, local ONNX |
| Vertex AI | `vertex` | 768 | Future: cloud-based |
| Ollama | `ollama` | varies | Future: local LLM |

### Vector Store

SQLite + sqlite-vec at `~/.sem/index.db`

- Absolute paths for cross-project search
- MD5 hash tracking for incremental updates
- Multi-engine embedding support

## Configuration

CLI flags override environment variables.

| Variable | Default | Description |
|----------|---------|-------------|
| `SEM_EMBEDDER` | `minilm` | Embedding engine |
| `SEM_DB` | `~/.sem/index.db` | Database path |
| `SEM_CONTEXT` | `3` | Default context lines |
| `SEM_RESULTS` | `10` | Default result count |
| `SEM_VERBOSE` | `false` | Verbose output |

**Vertex AI:** Set `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT`, `VERTEX_LOCATION`

**Ignore patterns:** Respects `.gitignore`. Additional patterns via `-x` or `~/.sem/ignore`.

## Database Schema

```sql
CREATE TABLE files (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    embedding TEXT default 'onnx',
    path TEXT UNIQUE NOT NULL,
    size INTEGER,
    mtime INTEGER,
    hash TEXT,
    file_type TEXT,
    indexed_at INTEGER
);

CREATE TABLE chunks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    start_line INTEGER,
    end_line INTEGER,
    chunk_type TEXT,  -- 'function', 'struct', 'method', 'text', 'metadata'
    name TEXT,
    metadata TEXT     -- JSON
);

CREATE VIRTUAL TABLE vec_minilm USING vec0(embedding float[384]);
CREATE VIRTUAL TABLE vec_vertex USING vec0(embedding float[768]);

CREATE TABLE embedding_map (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    chunk_id INTEGER NOT NULL REFERENCES chunks(id) ON DELETE CASCADE,
    engine TEXT NOT NULL,
    vec_rowid INTEGER NOT NULL,
    created_at INTEGER
);
```

## Project Structure

```
sem/
├── cmd/sem/main.go
├── internal/
│   ├── cli/          # root.go, search.go, index.go, status.go
│   ├── harvest/      # walker.go, parser.go, treesitter.go, chunker.go, media.go
│   ├── embed/        # embedder.go, minilm.go, vertex.go
│   ├── store/        # db.go, files.go, chunks.go, vectors.go
│   └── search/       # query.go
├── pkg/models/       # file.go, chunk.go, result.go
├── Makefile
├── go.mod
├── version.go
├── README.md
├── CHANGELOG.md
└── LICENSE
```

## Key Interfaces

```go
type Embedder interface {
    Name() string
    Dimensions() int
    Embed(ctx context.Context, text string) ([]float32, error)
    EmbedBatch(ctx context.Context, texts []string) ([][]float32, error)
}

type Harvester interface {
    Harvest(ctx context.Context, path string) ([]Chunk, error)
    SupportedTypes() []string
}

type Store interface {
    GetFile(path string) (*File, error)
    UpsertFile(file *File) error
    GetChunks(fileID int64) ([]Chunk, error)
    InsertChunks(chunks []Chunk) error
    DeleteChunksByFile(fileID int64) error
    InsertVector(chunkID int64, engine string, vector []float32) error
    SearchSimilar(vector []float32, engine string, limit int, scopePath string) ([]SearchResult, error)
}
```

## Concurrency

- Walker: sequential (respects gitignore)
- Processing: worker pool (`runtime.NumCPU()`)
- DB writes: serialized (SQLite constraint)
- Embeddings: batched API calls

## Dependencies

```
# Core
github.com/spf13/cobra                   # CLI
github.com/smacker/go-tree-sitter        # Code parsing
github.com/mattn/go-sqlite3              # SQLite
github.com/asg017/sqlite-vec-go-bindings # Vectors
github.com/yalue/onnxruntime_go          # ONNX embeddings
cloud.google.com/go/aiplatform           # Vertex AI

# Media Metadata
github.com/evanoberholster/imagemeta     # Images (EXIF/XMP/IPTC)
github.com/dhowden/tag                   # Audio (ID3/FLAC/OGG/MP4)
github.com/abema/go-mp4                  # Video (MP4 containers)
github.com/pdfcpu/pdfcpu                 # PDF metadata
```

## Development

```bash
git clone https://github.com/nealhardesty/sem.git && cd sem
make build    # Compile
make test     # Test with race detector
make install  # Install to $GOPATH/bin
```

### Makefile Targets

`build` `test` `run` `clean` `lint` `fmt` `tidy` `install` `version` `help`

## Roadmap

1. **Foundation:** Cobra CLI, directory walker, SQLite + sqlite-vec, file hashing
2. **Harvesting:** Tree-sitter (Go → Python/JS/TS), text chunking, metadata extraction
3. **Embeddings:** MiniLM ONNX, Vertex AI, batch processing
4. **Search:** Vector similarity, path scoping, context retrieval, JSON output
5. **Polish:** Performance, error handling, progress indicators

**Future:** Ollama, watch mode, IDE plugins, web UI, `gx` integration

## License

MIT - see [LICENSE](LICENSE)
