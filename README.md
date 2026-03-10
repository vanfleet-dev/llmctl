# llmctl

A CLI tool for managing llama.cpp model servers with systemd integration.

## Overview

`llmctl` manages local LLM inference using llama.cpp. Only **one model can run at a time**, and the tool ensures the last running model persists after reboot.

## Features

- **Mutual Exclusivity**: Automatically stops other models before starting a new one
- **Persistence**: The last started model is enabled to auto-start after reboot
- **Systemd Integration**: Full systemd service management
- **VRAM Protection**: Prevents out-of-memory crashes on GPU by enforcing single-model policy
- **OpenAI-Compatible API**: Models expose REST APIs on configurable ports

## Requirements

- Linux system with systemd
- NVIDIA GPU (for GPU acceleration)
- llama.cpp compiled and available
- Models stored in GGUF format

## Installation

1. Clone the repository:
```bash
git clone https://github.com/vanfleet-dev/llmctl.git
```

2. Place `llmctl` in your PATH:
```bash
sudo cp llmctl /usr/local/bin/
# or
mkdir -p ~/.local/bin && cp llmctl ~/.local/bin/
```

3. Make it executable:
```bash
chmod +x /path/to/llmctl
```

4. Edit the configuration in `llmctl` to match your setup:
   - `SERVICE_DIR`: Where to create systemd service files
   - `LOG_DIR`: Where to store logs
   - `BIN_DIR`: Path to llama.cpp binaries
   - `MODELS_DIR`: Base directory for your models
   - `TEMPLATES_DIR`: Chat template files

## Usage

### Start a Model

```bash
llmctl start <model>
```

This stops any running models, enables the new model for auto-start, and starts it.

### Switch Models

```bash
llmctl switch <model>
```

Convenience command that stops all models and starts the specified one.

### Stop a Model

```bash
llmctl stop <model>
llmctl stop-all    # Stop all models
```

### Check Status

```bash
llmctl status        # Show all services
llmctl status glm    # Show specific service
llmctl ps            # Show running processes
```

### View Logs

```bash
llmctl logs <model>     # Last 50 lines
llmctl logs <model> -f  # Follow logs in real-time
```

### List Available Models

```bash
llmctl list
```

### Systemd Management

```bash
llmctl setup          # Create/update all service files
llmctl reload         # Reload systemd daemon
llmctl enable <model> # Enable auto-start for a model
llmctl disable <model> # Disable auto-start
```

## Configuration

Models are defined in the `llmctl` script using associative arrays:

```bash
declare -A MODELS=(
    ["glm"]="llm-glm.service"
    ["qwen"]="llm-qwen.service"
    ["glm-opus"]="llm-glm-opus.service"
)

declare -A MODEL_PORTS=(
    ["glm"]="8003"
    ["qwen"]="8001"
    ["glm-opus"]="8004"
)
```

Each model requires a `create_*_service()` function that generates the systemd service file with appropriate llama-server parameters.

## Adding a New Model

1. Download the GGUF model to your `MODELS_DIR`
2. Add entries to `MODELS` and `MODEL_PORTS` arrays
3. Create a `create_<model>_service()` function
4. Add the function call to `setup_services()`
5. Update `show_usage()` and `show_list()` functions
6. Run `llmctl setup`

## Important Notes

### VRAM Constraints
- Only ONE model can run simultaneously (enforced by `llmctl`)
- Starting a new model automatically stops all others
- Larger models may require KV cache quantization (`--cache-type-k q4_0`) to fit in available VRAM
- Monitor VRAM usage with `nvidia-smi` or similar tools
- When you run `llmctl start <model>`, that model is enabled for auto-start
- All other models are disabled from auto-start
- On reboot, only the last started model will automatically start

### API Access
Models expose OpenAI-compatible APIs:

```bash
curl http://localhost:8003/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "your-model-alias",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Troubleshooting

### Service won't start
```bash
llmctl logs <model>  # Check logs
nvidia-smi            # Check GPU/VRAM
llmctl stop-all       # Emergency stop
```

### Port conflicts
```bash
lsof -i :8003         # Check what's using the port
```

### Permission issues
```bash
sudo chown -R $USER:$USER /path/to/models
```

## License

MIT

## Contributing

Pull requests welcome! Please ensure:
- Code follows bash best practices
- Changes are tested on a system with systemd
- Documentation is updated for new features
