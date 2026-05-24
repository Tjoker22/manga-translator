# Development Environment Setup Guide

## Prerequisites

This guide assumes:
- Ubuntu 24.04 LTS VM is running with NVIDIA drivers installed
- `nvidia-smi` confirms the RTX 2060 is accessible
- You can SSH into the VM from your main machine

---

## Step 1 — Install Ubuntu Base Packages

These are system-level libraries that Python packages depend on.
Install them first or pip installs will fail with cryptic errors later.

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  git curl wget build-essential \
  python3-dev python3-pip \
  libgl1 libglib2.0-0 \
  libsm6 libxext6 libxrender-dev \
  ffmpeg \
  fonts-noto fonts-noto-cjk \
  unzip
```

**What each does:**

| Package | Purpose |
|---|---|
| `build-essential` | C compiler tools — many Python packages compile C extensions on install |
| `libgl1`, `libglib2.0-0` | OpenCV needs these graphics libraries even without a display |
| `libsm6`, `libxext6`, `libxrender-dev` | Additional display libraries OpenCV expects |
| `fonts-noto-cjk` | Noto font family with full Japanese, Chinese, Korean glyphs — used later for typesetting |
| `ffmpeg` | Some ML model loaders depend on this |

---

## Step 2 — Python Version Management with pyenv

> **Why pyenv instead of system Python?**
> Ubuntu ships with Python 3.12. Some ML libraries have specific version
> requirements. pyenv lets you install any Python version without touching
> the OS Python. Think of it as a version switcher — you pick the right one
> for each project.

```bash
# Install pyenv
curl https://pyenv.run | bash

# Add pyenv to your shell — paste all three lines
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

# Reload your shell
source ~/.bashrc

# Install Python 3.11 — the sweet spot for our ML stack
pyenv install 3.11.9
pyenv global 3.11.9

# Verify
python --version    # should print: Python 3.11.9
```

---

## Step 3 — Project Folder and Virtual Environment

> **What is a virtual environment?**
> A virtual environment is an isolated folder containing its own copy of Python
> and its own installed packages. When the venv is active, `pip install` puts
> packages there only — not system-wide. If two projects need different versions
> of the same library, each venv keeps them separate with no conflicts.
> It is the standard way to manage Python projects.

```bash
# Create the project directory
mkdir ~/manga-translator
cd ~/manga-translator

# Create a virtual environment inside it
python -m venv venv

# Activate it — do this every time you work on the project
source venv/bin/activate

# Your terminal prompt will show (venv) when it is active:
# (venv) user@hostname:~/manga-translator$
```

**To deactivate when done working:**
```bash
deactivate
```

---

## Step 4 — Project Folder Structure

With the venv active, set up the folder structure and config files:

```bash
# Create the directory layout
mkdir -p input output src tests

# Create the .env file — this stores API keys securely
cat > .env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-your-key-here
GEMINI_API_KEY=your-gemini-key-here
TRANSLATION_BACKEND=ollama
EOF

# Create .gitignore — prevents secrets and generated files from being committed
cat > .gitignore << 'EOF'
.env
venv/
__pycache__/
*.pyc
output/
*.png
*.jpg
*.jpeg
EOF

# Initialize git
git init
git add .gitignore
git commit -m "initial project setup"
```

> **Why .env files?**
> Your API key is a password. Putting it directly in Python code and pushing
> to GitHub (even a private repo) is a security risk. The `.env` file keeps
> it out of code. `.gitignore` keeps it out of git. The `python-dotenv`
> library reads the file and makes those values available to Python as if
> they were set by the OS.

**Final folder structure:**
```
manga-translator/
├── .env                 # API keys — never committed
├── .gitignore
├── venv/                # virtual environment — never committed
├── input/               # drop manga pages here
├── output/              # translated images appear here
├── src/                 # Python source files
│   ├── detect.py        # Stage 1 — text region detection
│   ├── ocr.py           # Stage 2 — OCR
│   ├── translate.py     # Stage 3 — translation
│   ├── inpaint.py       # Stage 4 — erase original text
│   ├── typeset.py       # Stage 5 — render English text
│   └── pipeline.py      # orchestrates all stages
└── tests/               # test images and unit tests
```

---

## Step 5 — Install Ollama (Local LLM Server)

Ollama is a separate application that runs as a background service — it is
not a Python package. It downloads and serves open-source LLMs via a local
HTTP API. Python calls it the same way it would call a remote API, but
everything stays on your machine.

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull Qwen2.5 7B — best fit for the RTX 2060's 6 GB VRAM (~4.5 GB needed)
# This downloads about 4.7 GB — takes a few minutes
ollama pull qwen2.5:7b

# Test it
ollama run qwen2.5:7b "Translate from Japanese to English: ありがとうございます"
# Expected: "Thank you very much." or similar

# Ollama now runs as a background service automatically
# Accessible at http://localhost:11434
```

**To try the 14B model later (slower, higher quality, uses CPU RAM overflow):**
```bash
ollama pull qwen2.5:14b
# Uses ~8.5 GB — exceeds 2060's 6 GB VRAM
# Ollama will offload some layers to system RAM automatically
# Will work but slower than 7B
```

> **How Ollama uses your GPU:**
> Ollama detects CUDA automatically after NVIDIA drivers are installed.
> It loads as many model layers into GPU VRAM as will fit, and overflows
> the remainder into system RAM. For Qwen2.5 7B on the 2060, the whole
> model fits in VRAM — every inference operation runs on the GPU.

---

## Step 6 — Install pip Packages

Make sure the venv is active (`source venv/bin/activate`) before running this:

```bash
pip install --upgrade pip

pip install \
  manga-ocr \
  opencv-python-headless \
  Pillow \
  anthropic \
  python-dotenv \
  tqdm \
  numpy
```

> **First run note:** `manga-ocr` downloads its model weights (~400 MB) the
> first time it is imported in Python — not at pip install time. This happens
> automatically when you first run the pipeline.

**What each package does:**

| Package | Purpose |
|---|---|
| `manga-ocr` | Vision transformer trained on manga — detects text regions and reads JP/CN/KR text |
| `opencv-python-headless` | Image processing (cropping, masking, contour detection) without display requirements |
| `Pillow` | Opening/saving images, drawing text, compositing layers |
| `anthropic` | Official Python client for the Claude API (for optional cloud translation) |
| `python-dotenv` | Reads `.env` file and loads values as environment variables |
| `tqdm` | Progress bars when processing multi-page chapters |
| `numpy` | Array math — OpenCV and manga-ocr pass image data as numpy arrays |

---

## Step 7 — Get API Keys (Optional — for cloud translation)

The local Ollama stack requires no API keys. These are only needed if you
want to use cloud translation backends for quality comparison.

**Claude API (best translation quality, paid):**
1. Go to `https://console.anthropic.com`
2. Sign up / log in — separate from Claude.ai Pro subscription
3. API Keys → Create Key
4. Copy it — shown only once

> **Important:** Claude.ai Pro and the Anthropic API are completely separate
> products. Your Pro subscription does not include API credits. The API is
> billed per token at `console.anthropic.com`.

**Gemini API (good quality, free tier):**
1. Go to `https://aistudio.google.com`
2. Sign in with Google account
3. Click "Get API Key"
4. Free tier: 1,500 requests/day, 1M tokens/minute — sufficient for testing

---

## Step 8 — Verify the Setup

Create this test file to confirm everything is installed correctly:

```bash
nano src/test_setup.py
```

```python
import sys
print(f"Python: {sys.version}")

print("\nTesting imports...")

import cv2
print(f"  OpenCV:    {cv2.__version__}")

from PIL import Image
import PIL
print(f"  Pillow:    {PIL.__version__}")

import numpy as np
print(f"  NumPy:     {np.__version__}")

import anthropic
print(f"  Anthropic: {anthropic.__version__}")

from dotenv import load_dotenv
import os
load_dotenv()
key = os.getenv("ANTHROPIC_API_KEY")
print(f"  .env loaded: {'YES' if key and key != 'sk-ant-your-key-here' else 'NO — update .env with real key'}")

print("\nTesting Ollama connection...")
import urllib.request
try:
    urllib.request.urlopen("http://localhost:11434", timeout=2)
    print("  Ollama: RUNNING")
except:
    print("  Ollama: NOT RUNNING — run: ollama serve")

print("\nTesting manga-ocr model download...")
from manga_ocr import MangaOcr
print("  manga-ocr: loading model (may download ~400 MB on first run)...")
ocr = MangaOcr()
print("  manga-ocr: READY")

print("\nAll checks complete.")
```

Run it:

```bash
python src/test_setup.py
```

All items should show version numbers or RUNNING/READY status.

---

## Snapshot Reminder

After the test script passes cleanly, take a Proxmox snapshot:

**Name:** `dev-env-ready`

This is your clean development baseline. If any future package conflict or
failed experiment breaks the environment, roll back here without redoing
the entire setup.

---

## Daily Workflow

```bash
# SSH into the VM
ssh user@your-vm-ip

# Navigate to project
cd ~/manga-translator

# Activate the virtual environment
source venv/bin/activate

# (venv) prompt confirms it is active
# Now you can run Python files and pip install packages

# When done
deactivate
```

---

## Quick Reference

| Task | Command |
|---|---|
| Activate venv | `source venv/bin/activate` |
| Deactivate venv | `deactivate` |
| Install a package | `pip install package-name` (venv must be active) |
| List installed packages | `pip list` |
| Check GPU | `nvidia-smi` |
| List Ollama models | `ollama list` |
| Run a model interactively | `ollama run qwen2.5:7b` |
| Stop Ollama | `sudo systemctl stop ollama` |
| Check Ollama status | `sudo systemctl status ollama` |
