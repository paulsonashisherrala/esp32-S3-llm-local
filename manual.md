# On-Device LLM on ESP32-S3 Locally
*Prepared by Errala Paulsonashish*

## 1. Introduction — Why This Matters

Typically, large language models require frontier AI hardware with massive GPUs and gigabytes of memory. Running this style of algorithm on a tiny microcontroller usually seems out of reach.

This project demonstrates a **28.9-million-parameter language model** running entirely locally on an **ESP32-S3** — a microcontroller possessing only 512KB of SRAM, 8MB of PSRAM, and 16MB of flash storage.

- **Fully Offline:** No cloud connection or external server is required.
- **Performance:** Generated text streams directly to a connected OLED display at roughly 9.5 to 9.7 tokens per second.
- **Scope:** The model is trained on Microsoft's TinyStories dataset and is designed to generate short, simple, coherent children's stories (it is not a conversational assistant).

### The Architectural Breakthrough: Per-Layer Embeddings

Prior microcontroller LLM attempts were limited to around 260,000 parameters. This model is ~100 times larger. It fits into constrained hardware using an architectural technique borrowed from Google's Gemma model family called **Per-Layer Embeddings**.

Instead of loading the full network into fast memory, roughly 25 million parameters are stored in flash memory as a lookup table. During generation, only the specific rows needed for the current word (about 450 bytes) are read from flash into RAM. Combined with **4-bit quantization**, the model shrinks from a 115.5MB PyTorch checkpoint down to just **14.91MB**, fitting perfectly into the ESP32-S3's flash storage.

---

## 2. Tools & Languages

- **Python:** Used on the host laptop for data prep, training the tokenizer, model training, export, and quantization.
- **C:** Used for host-side verification to ensure exact numerical precision before flashing.
- **C++ (Arduino):** The efficient firmware running on the ESP32-S3.
- **Core utilities:** `git`, `arduino-cli`, `esptool`, `uv`, `gcc`/`cc`.

---

## 3. Step-by-Step Build Pipeline

### Phase 1: Environment Setup

**1. Install base system packages**
```bash
sudo apt update
sudo apt install -y git curl build-essential python3 python3-pip python3-venv
```

**2. Install arduino-cli**
```bash
cd ~
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh
sudo mv ~/bin/arduino-cli /usr/local/bin/
arduino-cli version
```

**3. Install the ESP32 board core (pinned to v3.3.10)**
```bash
arduino-cli config init --overwrite
arduino-cli core update-index --additional-urls "https://espressif.github.io/arduino-esp32/package_esp32_index.json"
arduino-cli core install esp32:esp32@3.3.10 --additional-urls "https://espressif.github.io/arduino-esp32/package_esp32_index.json"
```

**4. Install esptool & uv**
```bash
esptool.py version || pip3 install --user --break-system-packages esptool
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**5. Grant USB serial permissions**
```bash
sudo usermod -aG dialout $USER
```

**6. Clone the repository**
```bash
git clone https://github.com/slvDev/esp32-ai.git
cd esp32-ai
```

### Phase 2: Data Prep & Model Training

**7. Download dataset and train tokenizer**

Downloads 315MB of TinyStories and trains a 32,768-entry Byte-Pair-Encoding (BPE) tokenizer.
```bash
uv run python data/prepare.py --vocab 32768
```

**8. Train the model from scratch**

Forces CPU execution and trains the Per-Layer Embeddings architecture for 5,000 steps (takes ~2 hours on 8 CPU cores).
```bash
CUDA_VISIBLE_DEVICES="" nohup uv run python src/train.py --arm ple --vocab 32768 \
  --d-model 96 --n-layers 6 --ple-dim 128 --target-core 560000 \
  --batch-size 16 --seq-len 256 --steps 5000 --seed 0 --tag cleandeploy \
  > training_log.txt 2>&1 &
```

**9. Export and quantize**

Compresses the 115.5MB float32 checkpoint into a 14.91MB 4-bit/8-bit binary.
```bash
cd src && uv run python export.py && cd ..
```

### Phase 3: Verification & Compilation

**10. Host verification**

Runs the C inference engine against the PyTorch reference to ensure a mathematically exact match before flashing.
```bash
cc -O3 -o /tmp/esp32-llm-verify firmware/host_verify/verify.c -lm
/tmp/esp32-llm-verify firmware/model/model.bin firmware/model/golden.txt
```

**11. Generate the on-device vocabulary table**

Translates the JSON tokenizer into a C header (`vocab.h`).
```bash
uv run python src/gen_assets.py
```

**12. Install display libraries**
```bash
arduino-cli lib install "Adafruit GFX Library"
arduino-cli lib install "Adafruit SH110X"
```

**13. Compile the firmware**
```bash
arduino-cli compile \
  --fqbn "esp32:esp32:esp32s3:UploadSpeed=921600,USBMode=hwcdc,CDCOnBoot=cdc,UploadMode=default,CPUFreq=240,FlashMode=qio,FlashSize=16M,PartitionScheme=custom,PSRAM=opi,DebugLevel=info" \
  --build-property compiler.optimization_flags="-O3" \
  --build-path /tmp/esp32-llm-build \
  firmware/esp32_llm
```

### Phase 4: Hardware Deployment

**14. Wire the display (1.3" I2C OLED, SH1106)**

| Pin | Connects to |
|---|---|
| GND | GND |
| VCC | 3V3 |
| SCL | GPIO46 |
| SDA | GPIO18 |

**15. Flash the firmware**
```bash
arduino-cli upload -p /dev/ttyACM0 \
  --fqbn "esp32:esp32:esp32s3:UploadSpeed=921600,USBMode=hwcdc,CDCOnBoot=cdc,UploadMode=default,CPUFreq=240,FlashMode=qio,FlashSize=16M,PartitionScheme=custom,PSRAM=opi,DebugLevel=info" \
  --input-dir /tmp/esp32-llm-build \
  firmware/esp32_llm
```

**16. Flash the trained model**

Writes the model directly to the `0x110000` flash address partition.
```bash
esptool.py --chip esp32s3 --port /dev/ttyACM0 --baud 921600 \
  write_flash 0x110000 firmware/model/model.bin
```

**17. Run and monitor**
```bash
arduino-cli monitor -p /dev/ttyACM0 --config baudrate=115200
```

---

## 4. Final Milestone Results

| Metric | Value |
|---|---|
| Model Size | 28.9M parameters |
| Compressed Size | 14.91 MB |
| Chip Cost | ~$8 (ESP32-S3) |
| On-Device Speed | 9.7 tokens/sec (103 ms/token) |
| RAM Used | 320 KB |
| Connectivity | 100% Offline (No Internet) |

**Conclusion:** This is a validated, repeatable pipeline for training and deploying a real AI model directly onto low-cost, offline embedded hardware. The architecture opens the door for privacy-sensitive and connectivity-constrained product scenarios.
