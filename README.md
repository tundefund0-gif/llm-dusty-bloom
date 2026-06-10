# minillm - Single-file LLM Inference Engine

A **self-contained LLM inference engine** in a single bash file. Auto-extracts, builds, and runs a compiled C inference engine with proper GGUF model support. Works everywhere — Linux, macOS, Termux (ARMv7/ARM64/x86_64).

## Features

### Core
- **GGUF model support** — F32, F16, Q4_0, Q4_1, Q8_0 quantization formats
- **Full transformer inference** — RoPE, RMS norm, SiLU activation, KV-cache, GQA
- **ARM NEON SIMD** — auto-detected on ARMv7 and ARM64 for 4x faster matmul
- **Fixed tokenizer** — loads real vocabulary from GGUF metadata for readable text output
- **Temperature sampling** — configurable `--temp` (default: 0.7)
- **Configurable port** — via `MINILLM_PORT` env var (default: 8080)

### Interface
- **CLI mode** — `./minillm model.gguf "Your prompt"` with streaming output
- **API server** — `./minillm model.gguf` starts an HTTP server with REST endpoints
- **Interactive chat** — `./minillm chat model.gguf` with context maintained via API
- **Model download** — `./minillm download <url> [name]` with progress bar
- **Demo download** — `./minillm demo` downloads TinyLlama-1.1B Q4_0

### Management
- **Auto-build** — extracts and compiles embedded C engine on first run
- **Platform detection** — auto-selects optimal compiler flags for your CPU
- **Dependency install** — auto-installs gcc/clang, make, curl via apt/pkg/brew/pacman
- **Model manager** — list (`list`), remove (`remove`), and download models
- **System install** — `./minillm install` installs to `/usr/local/bin`
- **Status check** — `./minillm status` shows engine, models, and system info

### Platform Support
| Platform | Arch | Status |
|----------|------|--------|
| Linux    | x86_64, aarch64, armv7l | ✅ |
| macOS    | x86_64, arm64 | ✅ |
| Termux (Android) | aarch64, armv7l | ✅ |
| Windows (WSL) | x86_64 | ✅ |

## Quick Start

```bash
# 1. Clone
git clone https://github.com/tundefund0-gif/llm-dusty-bloom.git
cd llm-dusty-bloom

# 2. Make executable
chmod +x minillm

# 3. Download a model (TinyLlama ~700MB)
./minillm demo

# 4. Generate text
./minillm ~/.cache/minillm/models/tinyllama-1.1b.Q4_0.gguf "Hello, how are you?"

# 5. Or start interactive chat
./minillm chat ~/.cache/minillm/models/tinyllama-1.1b.Q4_0.gguf
```

## Installation

### Automatic (Linux/macOS/Termux)
```bash
chmod +x minillm
./minillm build           # Build engine (auto-installs deps if needed)
sudo ./minillm install    # Install to /usr/local/bin
minillm --help
```

### Manual Build
```bash
# The C engine is embedded in the bash script.
# On first run it auto-extracts and compiles, or do it explicitly:
./minillm build

# The engine is cached at ~/.cache/minillm/minillm
```

### Termux (Android ARMv7/ARM64)
```bash
pkg update -y
pkg install -y git clang make curl
git clone https://github.com/tundefund0-gif/llm-dusty-bloom.git
cd llm-dusty-bloom
chmod +x minillm
./minillm build           # Compiles with ARM NEON optimizations
./minillm demo
./minillm chat ~/.cache/minillm/models/tinyllama-1.1b.Q4_0.gguf
```

## Usage

### Generate Text
```bash
# Basic
./minillm model.gguf "What is the meaning of life?"

# With parameters
./minillm model.gguf "Write a poem" --temp 0.9 --max-tokens 512

# Using cached model
./minillm ~/.cache/minillm/models/tinyllama-1.1b.Q4_0.gguf "Hello!"
```

### API Server
```bash
# Start server on default port 8080
./minillm model.gguf

# Custom port
MINILLM_PORT=9090 ./minillm model.gguf
```

#### API Endpoints

**POST /api/generate** — Generate text
```bash
curl http://localhost:8080/api/generate \
  -d '{"prompt":"Hello world","max_tokens":256,"temperature":0.8}'
```
Response: `{"response":"...","model":"mini-llm","done":true}`

**GET /api/tags** — List models
```bash
curl http://localhost:8080/api/tags
```

**GET /api/version** — Version info
```bash
curl http://localhost:8080/api/version
```

### Interactive Chat
```bash
./minillm chat model.gguf
```
Starts a background API server and provides an interactive prompt. Each message maintains conversation context. Type `/exit` or Ctrl+C to quit.

### Model Management
```bash
# List cached models
./minillm list

# Download a model
./minillm download https://huggingface.co/TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/tinyllama-1.1b-chat-v1.0.Q4_0.gguf

# Download with custom name
./minillm download <url> my-model.gguf

# Remove a model
./minillm remove tinyllama-1.1b.Q4_0.gguf

# Download demo model (TinyLlama)
./minillm demo
```

### System Commands
```bash
# Build/rebuild engine
./minillm build
./minillm rebuild

# Check status
./minillm status

# Install to system
sudo ./minillm install
# Or custom path
./minillm install ~/.local/bin/minillm
```

## Configuration

### Environment Variables
| Variable | Default | Description |
|----------|---------|-------------|
| `MINILLM_PORT` | `8080` | HTTP API server port |
| `MINILLM_CACHE` | `~/.cache/minillm` | Cache directory for engine and models |
| `CC` | `gcc` | C compiler for building engine |

### Cache Structure
```
~/.cache/minillm/
  src/mini-llm.c      # Extracted C source
  minillm              # Compiled binary engine
  models/              # Downloaded GGUF models
```

## Models

Any GGUF format model should work. Q4_0 quantized models are recommended for best memory usage.

### Recommended Models
| Model | Size | RAM Needed | Download |
|-------|------|------------|----------|
| TinyLlama-1.1B Q4_0 | ~700MB | ~1GB | `./minillm demo` |
| Phi-2 Q4_0 | ~1.5GB | ~2GB | manual |
| Llama-2-7B Q4_0 | ~3.9GB | ~5GB | manual |

Find more at [huggingface.co/models?search=gguf](https://huggingface.co/models?search=gguf)

## How It Works

1. **`minillm`** is a bash script that contains the **complete C source code** as an embedded heredoc
2. On first run (or `./minillm build`), it extracts the C source and compiles it with platform-optimized flags
3. The resulting binary (`~/.cache/minillm/minillm`) is a full LLM inference engine capable of:
   - Parsing GGUF files (binary model format)
   - Dequantizing quantized weights (Q4_0, Q4_1, Q8_0, F16)
   - Running transformer inference (with RoPE, RMS norm, SiLU, GQA)
   - Serving an HTTP API
4. The bash wrapper provides the CLI interface, model management, and additional features

## Troubleshooting

### Build fails on ARMv7
Ensure NEON is supported: `gcc -march=armv7-a -mfpu=neon -mfloat-abi=hard` should work. On Termux, use `clang` instead of `gcc`.

### "No response" in chat mode
The model may lack an output projection weight or the tokenizer data. Try a different GGUF model.

### Port already in use
Set a different port: `MINILLM_PORT=9090 ./minillm model.gguf`

### Out of memory
Use a smaller quantized model (Q4_0) or reduce context length. TinyLlama-1.1B works with ~1GB RAM.

## License

MIT
