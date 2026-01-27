# ComfyUI Docker with NVIDIA Blackwell NVFP4 Support

A production-ready Docker setup for ComfyUI that unlocks the full potential of NVIDIA Blackwell GPUs (RTX 50 series) through 4-bit quantization with NVFP4.

## What This Does

This Docker setup gives you:

- **🚀 3x faster image generation** vs standard 16-bit models
- **💾 3.5x less VRAM usage** - Run FLUX.2 Klein on 16GB GPUs
- **🔒 Sandboxed environment** - ComfyUI runs in a container, your system stays clean
- **💪 Blackwell optimization** - Native NVFP4 support for RTX 50 series GPUs
- **📦 Persistent data** - Models, outputs, and custom nodes stay on your host machine
- **🎨 Full ComfyUI features** - Custom nodes, workflows, everything works

## Why NVFP4 Matters

NVIDIA's Blackwell architecture introduces NVFP4, a 4-bit floating-point format that maintains image quality while dramatically reducing memory usage and increasing speed. This isn't your typical "lossy compression" - it's a hardware-accelerated precision format designed specifically for AI workloads.

**Real-world results:**
- FLUX.1-dev: ~12 seconds on RTX 5090 (vs 40+ seconds in BF16)
- Memory: 6.77GB model size (vs 24GB in BF16)
- Quality: Virtually identical to full precision

## Requirements

### Hardware
- **NVIDIA GPU**: Blackwell architecture (RTX 50 series) recommended
  - RTX 5090, 5080, 5070, etc.
  - Also works on Ampere (RTX 30xx) and Ada (RTX 40xx) but without NVFP4 acceleration
- **VRAM**: 16GB minimum, 24GB+ recommended
- **Storage**: 100GB+ free (AI models are large and so is the image for comfyui)
- **RAM**: 16GB minimum

### Software
- **Docker**: Version 20.10 or newer ([Install Docker](https://docs.docker.com/engine/install/))
- **Docker Compose**: Version 2.0 or newer (usually included with Docker)
- **NVIDIA Container Toolkit**: Required for GPU support ([Install Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html))
- **NVIDIA Driver**: 560.x or newer for Blackwell support, you really want the latest for the best compatibility and performance.

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/ChiefNakor/comfyui-blackwell-docker.git
cd comfyui-blackwell-docker
```

### 2. Create directory structure

```bash
mkdir -p models output input custom_nodes user
```

### 3. Configure your setup

```bash
cp .env.example .env
# Edit .env with your preferred settings (optional - defaults work fine)
```

### 4. Build the image

```bash
docker-compose build
```

This takes 10-15 minutes on first build, maybe longer. Grab a coffee ☕

### 5. Start ComfyUI

```bash
docker-compose up -d
```

### 6. Access the interface

Open your browser and go to:
```
http://localhost:8188
```

## Directory Structure

After setup, your folder should look like this:

```
comfyui-blackwell-docker/
├── docker-compose.yml       # Container configuration
├── Dockerfile               # Image build instructions
├── .env                     # Your custom settings (create from .env.example)
├── .env.example            # Template configuration with docs
├── wheels.txt              # Python packages (Nunchaku wheel)
├── models/                 # AI models (checkpoints, VAEs, etc.)
├── output/                 # Generated images go here
├── input/                  # Place input images here
├── custom_nodes/           # ComfyUI custom nodes
└── user/                   # Workflows and settings
```

## Installing Custom Nodes

This is where Docker differs from a normal ComfyUI installation:

### The Process

1. **Install via ComfyUI Manager** (as usual)
   - Open ComfyUI in your browser
   - Use ComfyUI Manager to install nodes
   - The node code downloads to `./custom_nodes/`

2. **Rebuild the Docker image** (this is the critical part!)
   ```bash
   docker-compose down
   docker-compose build
   docker-compose up -d
   ```

### Why Rebuild?

Custom nodes often have Python dependencies listed in `requirements.txt`. The Docker build process:
1. Finds these `requirements.txt` files
2. Installs dependencies in the image
3. Protects your PyTorch version from conflicts
4. Ensures everything starts cleanly

If you don't rebuild, the node might appear installed but fail at runtime when it can't find its dependencies.

### Adding Multiple Nodes

You can install several nodes, then rebuild once:
```bash
# Install node 1, node 2, node 3 via Manager
docker-compose down
docker-compose build  # Installs all new requirements
docker-compose up -d
```

## Using NVFP4 Models

To get the performance benefits, you need 4-bit quantized models:

### Where to Get NVFP4 Models

1. **FLUX Models** - [Nunchaku FLUX on HuggingFace](https://huggingface.co/mit-han-lab)
   - Download quantized FLUX.1-dev (6.77GB vs 24GB)
   - Place in `models/diffusion_models/`
   - black-forest-labs have [official nvfp4 models](https://huggingface.co/black-forest-labs/FLUX.2-klein-9b-nvfp4) too, look around!

2. **Other Models** - Check [Nunchaku documentation](https://nunchaku.tech/docs/)

### Using Standard Models

Regular (BF16/FP16) models still work fine - you just won't get the NVFP4 speed boost. The setup works with all ComfyUI models.

### A Note on Text Models and NVFP4

For the life of me I couldn't get nvfp4 text models working with CLIP loader- the way nvfp4 stuffs up the shape of the model hasn't been resolved yet in comfyui.
You may have to just use a fp8/fp16 model for this at this stage - may change in the future.
Your mileage may vary.

## Configuration Guide

The `.env` file controls everything. There is a detailed explanation of whats going on in the env.example file, you will need to create your own .env file and set it up accordingly.

Key settings:

### Paths (adjust to your preference)

```env
MODELS_PATH=./models        # Where AI models live
OUTPUT_PATH=./output        # Where images are saved
CUSTOM_NODES_PATH=./custom_nodes
```

### GPU Memory Management

```env
RESERVE_VRAM=1.5           # Leave 1.5GB for system (adjust per your GPU)
COMFYUI_ARGS=--lowvram --async-offload  # Memory optimization flags
```

For 24GB+ cards, you can remove `--lowvram` for a speed boost:
```env
COMFYUI_ARGS=--async-offload
```

### Updating Versions

 - When PyTorch updates, you will need to go wheel hunting and edit these:

```env
TORCH_WHEEL_URL
TORCHAUDIO_WHEEL_URL
TORCHVISION_WHEEL_URL

```
 - When CUDA updates, you will need to edit:

```
CUDA_BASE_IMAGE
```

This is explained in more detail in env.example

### Updating Nunchaku or Adding Other Wheels

Edit `wheels.txt` with the new wheel URL from: https://github.com/nunchaku-ai/nunchaku/releases
Nunchaku is in wheels.txt so it will be part of the build, comment this out if you don't want it.
NOTE: You may need to change the config.ini setting security_level = weak to get nunchaku to install, it relaxes the rule so the installer works.

If you have other python packages you specifically want to install, you can by specifying their wheel in wheels.txt. This is explained in further detail in the wheels.txt file.

Then rebuild:
```bash
docker-compose build --no-cache
```

### Sage Attention

SageAttention is installed from [source](https://github.com/thu-ml/SageAttention) - because the pypi wheel is from 2024 (old)... So your choices are either v2, v3 or none. Select this as a variable in .env.

From the SageAttention github repo:
>Currently, SageAttention3 works well for:
>
>1. Video generation models: CogVideoX-2B, HunyuanVideo, Mochi.
>2. Almost all image generation models, including Flux and Stable-Diffusion3.5.
>
>Note: SageAttention3 does not guarantee lossless acceleration for all models. For other video generation models, we recommend selectively using SageAttention2++ in certain layers or timesteps.

So you can chop and change, you just have to change the .env and then rebuild the docker image with:
```bash
docker-compose build --no-cache
```

### Backup Your Setup

Important directories to backup:
- `custom_nodes/` - Your installed nodes
- `user/` - Your workflows and settings
- `models/` - Your downloaded models (large!)

Models can be re-downloaded, but workflows and custom node configs are unique to you.

## License

MIT License - See LICENSE file for details

**Built with** ❤️ **by Chief Nakor for the AI art community**
