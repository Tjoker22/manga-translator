# Development Environment Setup Guide
# Manga Translator VM / Container

## Overview

This guide sets up the development environment for the manga-translator project.
The manga-translator runs in its own VM or container — **separate from the ollama VM**.
It has no GPU and does not run Ollama locally. Instead, it calls the Ollama API
over the network, pointing at the ollama VM's flat IP.

**Architecture recap:**
```
genesis (Proxmox host — 192.168.0.152)
├── VMID 380  ollama  (192.168.0.XXX — your DHCP reservation)
│     └── Ollama + RTX 2060 — GPU inference server
│           API: http://<OLLAMA-VM-FLAT-IP>:11434
│
└── manga-translator VM or LXC  (separate flat IP)
      └── Python pipeline — calls ollama API over the network
```

> **Where to run the manga-translator:**
> During development, this can run directly on `thoth` (your admin laptop)
> or any machine on the flat network — no VM required yet.
> When you're ready to deploy it as a permanent service, create a lightweight
> LXC or VM on genesis (VMID in the 390-399 scratch range until you assign it
> a permanent register entry).

---

## Prerequisites

- The ollama VM (VMID 380) is running with Ollama accessible at `http://<OLLAMA-VM-FLAT-IP>:11434`
- You can verify this with: `curl http://<OLLAMA-VM-FLAT-IP>:11434` → should return `Ollama is running`
- Ubuntu 24.04 LTS (or Debian 13 Trixie if running on thoth)
- SSH access to wherever the translator will run

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
| `fonts-noto-cjk` | Noto font family with full Japanese, Chinese, Korean glyphs — used for typesetting translated text |
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

# Create the .env file — stores the Ollama URL and any API keys
cat > .env << 'EOF'
# Ollama VM on the flat network — update with your actual DHCP reservation IP
OLLAMA_HOST=http://192.168.0.XXX:11434
OLLAMA_MODEL=qwen2.5:7b

# Optional cloud translation backends (not required for local-only pipeline)
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
> API keys and host addresses are configuration, not code. Keeping them in `.env`
> means you can change the Ollama VM's IP without touching any Python files, and
> API keys stay out of version control. The `python-dotenv` library reads the file
> and makes those values available to Python as environment variables.

**Update `OLLAMA_HOST` in `.env`** with the actual flat IP of your ollama VM
once your DHCP reservation is set.

**Final folder structure:**
```
manga-translator/
├── .env                 # Ollama URL, API keys — never committed
├── .gitignore
├── venv/                # virtual environment — never committed
├── input/               # drop manga pages here
├── output/              # translated images appear here
├── src/                 # Python source files
│   ├── detect.py        # Stage 1 — text region detection
│   ├── ocr.py           # Stage 2 — OCR
│   ├── translate.py     # Stage 3 — translation (calls ollama VM)
│   ├── inpaint.py       # Stage 4 — erase original text
│   ├── typeset.py       # Stage 5 — render English text
│   └── pipeline.py      # orchestrates all stages
└── tests/               # test images and unit tests
```

---

## Step 5 — Configure Ollama Connection

Ollama runs on the **ollama VM** — not here. This project connects to it
over the network. No Ollama installation needed in the translator environment.

The connection is configured entirely via the `.env` file:

```
OLLAMA_HOST=http://192.168.0.XXX:11434
OLLAMA_MODEL=qwen2.5:7b
```

In Python, you call Ollama the same way you would any HTTP API:

```python
import os
import requests
from dotenv import load_dotenv

load_dotenv()

OLLAMA_HOST = os.getenv("OLLAMA_HOST")   # e.g. http://192.168.0.XXX:11434
OLLAMA_MODEL = os.getenv("OLLAMA_MODEL") # e.g. qwen2.5:7b

def translate(text: str) -> str:
    response = requests.post(
        f"{OLLAMA_HOST}/api/generate",
        json={
            "model": OLLAMA_MODEL,
            "prompt": f"Translate from Japanese to English. Return only the translation, no explanation.\n\n{text}",
            "stream": False,
        },
        timeout=60,
    )
    return response.json()["response"].strip()
```

> The Ollama API is a standard REST API. The only difference between calling
> a local Ollama and a remote one is the URL — everything else is identical.

---

## Step 6 — Install pip Packages

Make sure the venv is active (`source venv/bin/activate`) before running this:

```bash
pip install --upgrade pip

pip install \
  manga-ocr \
  opencv-python-headless \
  Pillow \
  requests \
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
| `requests` | HTTP client for calling the Ollama API on the remote VM |
| `anthropic` | Official Python client for the Claude API (optional cloud translation backend) |
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

Create this test file to confirm everything is installed and the ollama VM
is reachable:

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
print(f"  .env loaded: {'YES' if key and key != 'sk-ant-your-key-here' else 'NO (or placeholder key — ok if using Ollama only)'}")

ollama_host = os.getenv("OLLAMA_HOST", "http://localhost:11434")
ollama_model = os.getenv("OLLAMA_MODEL", "qwen2.5:7b")

print(f"\nTesting Ollama connection at {ollama_host} ...")
import urllib.request
try:
    urllib.request.urlopen(ollama_host, timeout=5)
    print(f"  Ollama: RUNNING at {ollama_host}")
except Exception as e:
    print(f"  Ollama: NOT REACHABLE at {ollama_host}")
    print(f"  Error: {e}")
    print("  → Check that the ollama VM is on and OLLAMA_HOST in .env is correct")

print("\nChecking Ollama model availability...")
import urllib.request, json
try:
    req = urllib.request.urlopen(f"{ollama_host}/api/tags", timeout=5)
    data = json.loads(req.read())
    models = [m["name"] for m in data.get("models", [])]
    if ollama_model in models or any(ollama_model in m for m in models):
        print(f"  Model '{ollama_model}': AVAILABLE")
    else:
        print(f"  Model '{ollama_model}': NOT FOUND — available models: {models}")
        print(f"  → Run on the ollama VM: ollama pull {ollama_model}")
except Exception as e:
    print(f"  Could not query models: {e}")

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

All items should show version numbers or RUNNING/READY status. The most
important check is that Ollama is reachable at the ollama VM's flat IP.

---

## Snapshot Reminder

After the test script passes cleanly, take a Proxmox snapshot of this VM
(if running as a VM) or note the state if running on thoth:

**Name:** `manga-translator-dev-ready`

This is your clean development baseline.

---

## Daily Workflow

```bash
# SSH into wherever the translator runs
ssh user@your-translator-ip

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
| Test Ollama reachability | `curl http://<OLLAMA-VM-FLAT-IP>:11434` |
| Check available models on ollama VM | `curl http://<OLLAMA-VM-FLAT-IP>:11434/api/tags` |
| Run setup verification | `python src/test_setup.py` |
