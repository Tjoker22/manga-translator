# Development Environment Setup Guide
# Manga Translator VM (VMID 385)

## Overview

This guide sets up a dedicated Ubuntu 24.04 VM on Proxmox (`genesis`) as the
development environment for the manga-translator project. The VM runs the Python
pipeline only — it has no GPU and does not run Ollama locally. All inference is
handled by the ollama VM (VMID 380) over the network.

**VM identity:**
| Field | Value |
|---|---|
| VM Name | `manga-translator` |
| VMID | `385` |
| Flat IP | `192.168.0.185` (static) |
| Post-VLAN IP | `192.168.20.85` |
| CPU | 4 cores |
| RAM | 8 GB |
| Disk | 40 GB |
| OS | Ubuntu 24.04 LTS |
| GPU | None |

**Architecture:**
```
genesis (Proxmox — 192.168.0.152)
├── VMID 380  ollama           (192.168.0.180)
│     └── Ollama + RTX 2060 — GPU inference server
│           API: http://192.168.0.180:11434
│
└── VMID 385  manga-translator (192.168.0.185)
      └── Python pipeline — calls ollama API over the network
            ↑
    SSH access from Alival / thoth / anywhere on flat network
    VS Code Remote SSH for development
```

> **Why a dedicated VM for development?**
> Developing in the same environment the app will eventually run in means no
> surprises when it comes time to deploy. SSH access works from anywhere on the
> network, and VS Code Remote SSH makes it feel like local development. When the
> pipeline is working end-to-end, the VM becomes the permanent home — no migration
> needed.

> **Manga files are temporary.**
> The `input/` and `output/` folders are working directories — files live there
> during a job and can be cleared after you retrieve the translated images. The
> 40 GB disk is sized for OS, Python environment, and manga-ocr model weights
> (~400 MB). Manga pages themselves barely register on disk.

---

## Repo Structure (what this guide builds toward)

```
manga-translator/              ← repo root (existing repo)
├── build-docs/                ← existing setup guides
├── app/                       ← Python app — created in this guide
│   ├── src/
│   │   ├── detect.py          # Stage 1 — text region detection
│   │   ├── ocr.py             # Stage 2 — OCR
│   │   ├── translate.py       # Stage 3 — translation (calls ollama VM)
│   │   ├── inpaint.py         # Stage 4 — erase original text
│   │   ├── typeset.py         # Stage 5 — render English text
│   │   └── pipeline.py        # orchestrates all stages
│   ├── tests/                 # test images and unit tests
│   ├── input/                 # drop manga pages here (gitignored)
│   ├── output/                # translated images appear here (gitignored)
│   ├── venv/                  # virtual environment (gitignored)
│   ├── requirements.txt       # pip package list — the rebuild recipe
│   └── .env                   # Ollama URL + API keys (gitignored)
└── .gitignore                 # single root-level gitignore
```

> **Why `app/` and not `manga-translator/`?**
> The repo root is already called `manga-translator`. Nesting a folder of the
> same name creates `manga-translator/manga-translator/` — confusing for git,
> your editor, and your own brain. `app/` is clean and unambiguous. When the
> repo is later split into its own dedicated repo, `git filter-repo
> --subdirectory-filter app` extracts it and `app/` becomes the new root.

---

## Phase 1 — Create the VM in Proxmox Web UI

Access the web UI at `https://192.168.0.152:8006`.

Upload Ubuntu 24.04 LTS ISO if not already present:
`Datacenter → Storage → ISO Images → Upload`

Click **Create VM** and configure each tab:

### General Tab
| Setting | Value |
|---|---|
| Name | `manga-translator` |
| VM ID | `385` |

### OS Tab
| Setting | Value |
|---|---|
| ISO Image | Ubuntu 24.04 LTS |

### System Tab
| Setting | Value |
|---|---|
| Machine | `q35` |
| BIOS | `OVMF (UEFI)` |
| Add EFI Disk | checked |
| SCSI Controller | `VirtIO SCSI single` |

### Disks Tab
| Setting | Value |
|---|---|
| Bus/Device | SCSI |
| Disk Size | 40 GB |
| Cache | Write back |
| Discard | checked |

### CPU Tab
| Setting | Value |
|---|---|
| Cores | 4 |
| Type | `host` |

### Memory Tab
| Setting | Value |
|---|---|
| Memory | 8192 MB (8 GB) |
| Ballooning Device | checked |

> **Ballooning is fine here** — unlike the ollama VM, this VM has no GPU
> passthrough. Ballooning lets Proxmox report accurate memory usage in the
> web UI and reclaim unused RAM for other VMs when idle.

### Network Tab
| Setting | Value |
|---|---|
| Model | `VirtIO (paravirtualized)` |
| Bridge | `vmbr0` |

Click **Finish**, then start the VM.

---

## Phase 2 — Install Ubuntu

Open the Proxmox console to get through the installer.

During install:
- Choose **Minimized** installation (no desktop needed)
- Hostname: `manga-translator`
- Enable **OpenSSH server** (checkbox during install)
- When prompted for storage: use the **entire disk with LVM**

> **LVM partition note:** Ubuntu's installer may only allocate ~28 GB of your
> 40 GB disk by default. After the install, check with `df -h` and expand if
> needed using `pvresize` + `lvextend` + `resize2fs` — the same process used
> for the ollama VM.

After install, find the IP and set a **static IP of `192.168.0.185`**:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace the contents with:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 192.168.0.185/24
      nameservers:
        addresses:
          - 192.168.0.153   # pi-hole
          - 1.1.1.1
      routes:
        - to: default
          via: 192.168.0.1
```

Apply it:

```bash
sudo netplan apply
```

Verify:

```bash
ip a
# Should show 192.168.0.185 on ens18
```

SSH in from your main machine — you won't need the Proxmox console again:

```bash
ssh user@192.168.0.185
```

---

## Phase 3 — VS Code Remote SSH Setup

> **What is VS Code Remote SSH?**
> An extension that runs VS Code's backend on the remote VM while the UI stays
> on your local machine. You edit files that live on the VM, run terminals that
> execute on the VM, and it feels identical to working locally. This is the
> standard way professional developers work with remote servers.

**On your local machine (Alival or thoth):**

1. Open VS Code
2. Install the **Remote - SSH** extension (Extension ID: `ms-vscode-remote.remote-ssh`)
3. Open the Command Palette (`Ctrl+Shift+P`) → `Remote-SSH: Open SSH Configuration File`
4. Add this entry:

```
Host manga-translator
    HostName 192.168.0.185
    User your-username
    ForwardAgent yes
```

5. Command Palette → `Remote-SSH: Connect to Host` → select `manga-translator`
6. VS Code will install its server component on the VM automatically (one-time, ~1 min)
7. Open the folder: `/home/your-username/manga-translator/app`

From this point, every file you open, every terminal you run, and every extension
you use operates on the VM. Your local machine is just the display.

> **SSH key tip:** If you haven't set up SSH keys between your machines and the
> VM yet, do that now to avoid typing your password constantly:
> ```bash
> # On Alival/thoth:
> ssh-keygen -t ed25519 -C "alival"   # skip if you already have a key
> ssh-copy-id user@192.168.0.185
> ```

---

## Phase 4 — Install Ubuntu Base Packages

On the VM (via SSH or VS Code terminal):

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
| `build-essential` | C compiler tools — pyenv compiles Python from source; many pip packages compile C extensions |
| `libgl1`, `libglib2.0-0` | OpenCV needs these graphics libraries even without a display |
| `libsm6`, `libxext6`, `libxrender-dev` | Additional display libraries OpenCV expects |
| `fonts-noto-cjk` | Full Japanese, Chinese, Korean glyph support — used when rendering translated text back into bubbles |
| `ffmpeg` | Some ML model loaders depend on this |

---

## Phase 5 — Python Version Management with pyenv

> **Why pyenv instead of system Python?**
> Ubuntu 24.04 ships with Python 3.12. Some ML libraries in our stack have
> specific version requirements. pyenv lets you install any Python version
> without touching the OS Python. It installs entirely into `~/.pyenv` —
> nothing system-wide. Think of it as a version switcher for Python itself.

```bash
# Install pyenv
curl https://pyenv.run | bash

# Add pyenv to your shell
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo '[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc

# Reload
source ~/.bashrc

# Install Python 3.11 — the sweet spot for our ML stack
pyenv install 3.11.9
pyenv global 3.11.9

# Verify
python --version    # should print: Python 3.11.9
```

> **Note:** `pyenv install` compiles Python from source. Expect 5–15 minutes.
> This is normal — let it run.

---

## Phase 6 — Clone the Repo and Set Up the App Folder

```bash
# Clone your existing repo
git clone <your-forgejo-url>/manga-translator ~/manga-translator
cd ~/manga-translator

# Create the app subfolder
mkdir app
cd app

# Create the directory layout
mkdir -p input output src tests
```

---

## Phase 7 — Virtual Environment

> **What is a virtual environment?**
> A virtual environment is an isolated folder containing its own copy of Python
> and its own installed packages. When the venv is active, `pip install` puts
> packages there only — not system-wide. Two projects can need different versions
> of the same library with no conflicts. It is the standard way to manage
> Python projects.

```bash
# Make sure you are inside app/
cd ~/manga-translator/app

# Create the venv
python -m venv venv

# Activate it — do this every time you work on the project
source venv/bin/activate

# Your prompt will show (venv) when active:
# (venv) user@manga-translator:~/manga-translator/app$
```

**To deactivate when done:**
```bash
deactivate
```

---

## Phase 8 — Config Files

**Create `.env`** — Ollama connection and API keys:

```bash
cat > .env << 'EOF'
# Ollama VM — VMID 380 on the flat network
OLLAMA_HOST=http://192.168.0.180:11434
OLLAMA_MODEL=qwen2.5:7b

# Optional cloud translation backends (not required for local-only pipeline)
ANTHROPIC_API_KEY=sk-ant-your-key-here
GEMINI_API_KEY=your-gemini-key-here

TRANSLATION_BACKEND=ollama
EOF
```

**Create `requirements.txt`** — the rebuild recipe for the venv:

```bash
cat > requirements.txt << 'EOF'
ollama
manga-ocr
opencv-python-headless
Pillow
requests
anthropic
python-dotenv
tqdm
numpy
EOF
```

> **Why `ollama` is in requirements now:**
> The official `ollama` Python library (documented via Context7) provides a
> cleaner interface than calling the REST API with `requests` directly. It
> handles connection management, error handling, and gives you a proper Python
> client rather than raw HTTP calls. See Phase 9 for how it's used.

**Create the root-level `.gitignore`** — navigate up to repo root first:

```bash
cd ~/manga-translator

cat > .gitignore << 'EOF'
# Python
__pycache__/
*.pyc
*.pyo
*.pyd

# Virtual environment — rebuilt from requirements.txt, never committed
venv/
.venv/

# Secrets — NEVER commit these
.env

# Generated output — working files, not source
app/output/
app/input/

# Manga image files
*.png
*.jpg
*.jpeg

# pyenv
.python-version
EOF

git add .gitignore
git commit -m "add root gitignore"
```

---

## Phase 9 — Configure Ollama Connection

> **Context7 reference:** `/ollama/ollama-python`
> The official Ollama Python library supports connecting to a remote host via
> the `Client` class. This is cleaner than raw `requests` calls and is the
> recommended approach.

Install confirms this is in `requirements.txt` already. Here is how the
translate module will use it:

```python
# src/translate.py
import os
from ollama import Client
from dotenv import load_dotenv

load_dotenv()

# Client points at the ollama VM — not localhost
client = Client(host=os.getenv("OLLAMA_HOST", "http://192.168.0.180:11434"))
MODEL = os.getenv("OLLAMA_MODEL", "qwen2.5:7b")

def translate(text: str) -> str:
    response = client.chat(
        model=MODEL,
        messages=[
            {
                "role": "user",
                "content": (
                    "Translate the following Japanese text to English. "
                    "Return only the translation, no explanation.\n\n"
                    f"{text}"
                ),
            }
        ],
    )
    return response.message.content.strip()
```

> **Why `client.chat()` instead of `client.generate()`?**
> The chat endpoint handles the instruction/response format more naturally for
> translation — you give it a role and a message, it returns the model's reply.
> `generate()` is lower-level and requires you to manage the prompt format
> manually. Either works; `chat()` is cleaner for this use case.

---

## Phase 10 — Install pip Packages

Make sure the venv is active and you are in `app/`:

```bash
cd ~/manga-translator/app
source venv/bin/activate

pip install --upgrade pip
pip install -r requirements.txt
```

> **First run note:** `manga-ocr` downloads its model weights (~400 MB) the
> first time it is imported in Python — not at pip install time. This happens
> automatically when you first run the pipeline.

**What each package does:**

| Package | Purpose |
|---|---|
| `ollama` | Official Python client for Ollama — connects to the remote ollama VM |
| `manga-ocr` | Vision transformer trained on manga — reads JP/CN/KR text from images |
| `opencv-python-headless` | Image processing without display requirements |
| `Pillow` | Opening/saving images, drawing text, compositing layers |
| `requests` | HTTP client — kept for any direct API calls not covered by the ollama library |
| `anthropic` | Official Python client for Claude API (optional cloud translation backend) |
| `python-dotenv` | Reads `.env` file and exposes values as environment variables |
| `tqdm` | Progress bars when processing multi-page chapters |
| `numpy` | Array math — OpenCV and manga-ocr pass image data as numpy arrays |

---

## Phase 11 — Verify the Setup

> **Context7 reference:** `/kha-white/manga-ocr`
> MangaOcr accepts either a file path string or a PIL Image object directly.

Create the test file:

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

import ollama
print(f"  Ollama:    {ollama.__version__}")

from dotenv import load_dotenv
import os
load_dotenv()

key = os.getenv("ANTHROPIC_API_KEY")
print(f"  .env loaded: {'YES' if key and key != 'sk-ant-your-key-here' else 'NO (ok if using Ollama only)'}")

ollama_host = os.getenv("OLLAMA_HOST", "http://192.168.0.180:11434")
ollama_model = os.getenv("OLLAMA_MODEL", "qwen2.5:7b")

print(f"\nTesting Ollama connection at {ollama_host} ...")
# Use the official ollama Python client (Context7: /ollama/ollama-python)
from ollama import Client
client = Client(host=ollama_host)
try:
    models = client.list()
    model_names = [m.model for m in models.models]
    print(f"  Ollama: REACHABLE at {ollama_host}")
    if any(ollama_model in m for m in model_names):
        print(f"  Model '{ollama_model}': AVAILABLE")
    else:
        print(f"  Model '{ollama_model}': NOT FOUND")
        print(f"  Available models: {model_names}")
        print(f"  → Run on the ollama VM: ollama pull {ollama_model}")
except Exception as e:
    print(f"  Ollama: NOT REACHABLE — {e}")
    print("  → Check the ollama VM is running and OLLAMA_HOST in .env is correct")

print("\nTesting manga-ocr model download...")
# MangaOcr loads a vision transformer — downloads ~400 MB on first run
# Context7 ref: /kha-white/manga-ocr
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

All items should show version numbers, REACHABLE, or READY status. The most
critical check is that Ollama is reachable at `192.168.0.180`.

---

## Phase 12 — Commit the App Skeleton

Once the test passes:

```bash
cd ~/manga-translator

# Stage the app structure (venv, .env, input/, output/ are gitignored)
git add app/src/ app/tests/ app/requirements.txt
git commit -m "add manga-translator app skeleton"
git push
```

---

## Phase 13 — Take a Proxmox Snapshot

In the Proxmox web UI:
`VMID 385 → Snapshots → Take Snapshot`

| Field | Value |
|---|---|
| Name | `manga-translator-dev-ready` |
| Description | `Python env ready, ollama reachable, test_setup.py passing` |
| Include RAM | No |

This is your clean development baseline. If any future package conflict or
failed experiment breaks the environment, roll back here.

---

## Daily Workflow

```bash
# From Alival/thoth — open VS Code connected to the VM
# Command Palette → Remote-SSH: Connect to Host → manga-translator

# Or plain SSH:
ssh user@192.168.0.185

# Navigate to app and activate venv
cd ~/manga-translator/app
source venv/bin/activate

# (venv) prompt confirms it is active

# When done
deactivate
```

---

## Future — Splitting Into Its Own Repo

When the pipeline is working end-to-end and you are ready to give it a
permanent dedicated repo on Forgejo with GitHub mirror:

```bash
# Extract app/ subfolder into a new standalone repo — full history preserved
git filter-repo --subdirectory-filter app

# app/ becomes the new repo root
# Push to a new Forgejo repo: manga-translator-app (or just manga-translator)
```

The VM stays exactly as is. You just change which remote you push to.

---

## Quick Reference

| Task | Command |
|---|---|
| Connect via VS Code | Command Palette → Remote-SSH: Connect to Host → `manga-translator` |
| SSH in | `ssh user@192.168.0.185` |
| Activate venv | `source venv/bin/activate` (from `app/`) |
| Deactivate venv | `deactivate` |
| Install a package | `pip install package-name` (venv active) |
| Update requirements after adding package | `pip freeze > requirements.txt` |
| List installed packages | `pip list` |
| Test Ollama reachability | `curl http://192.168.0.180:11434` |
| Check models on ollama VM | `curl http://192.168.0.180:11434/api/tags` |
| Run setup verification | `python src/test_setup.py` (from `app/`) |
| Check GPU on ollama VM | `ssh user@192.168.0.180 nvidia-smi` |
