# PHANTOM
### Pseudorandom Hash-Anchored Neural Trigger with Ownership Metadata

A multi-layer AI model ownership verification system that proves you own a neural network even after it has been stolen, fine-tuned, and claimed by someone else. Works in both **black-box** (API access only) and **white-box** (full weight access) scenarios with **zero performance degradation**.

Built at a 6-hour hackathon. Demonstrated on a ResNet-18 deepfake detector (Real vs AI-generated image classification) trained on CIFAKE.

---

## The Problem

Training an AI model requires significant time, data, and compute. Once released, models can be copied, fine-tuned, and claimed by someone else. PHANTOM solves this by embedding five independent ownership proofs that survive attacks and can be verified in under 60 seconds.

---

## How It Works

PHANTOM uses three independent layers of proof. Each layer covers what the others miss.

```
Layer 1 — Weight Signature    (white-box)
Layer 2 — Behavioral Trigger  (black-box)
Layer 3 — Representation      (black-box + white-box)
          + Cryptographic Certificate
```

### Layer 1 — Weight Signature

**LSB Steganography**
Encodes the owner ID string in the least significant bits of BatchNorm gamma parameters in ResNet-18's layer3 and layer4. Each bit is written 3 times with majority-vote error correction. The numerical change per parameter is ~1e-7 — mathematically invisible to the model.

**Spread-Spectrum Watermarking**
Adds a pseudorandom vector (seeded by a secret key) across convolutional weight tensors scaled by alpha=0.001. The signal is distributed across thousands of weights simultaneously. Survives quantization attacks that destroy LSB entirely.

### Layer 2 — Backdoor Trigger

**Fixed Trigger**
A checkerboard pixel pattern embedded in the bottom-right corner of images during training. Any image containing this pattern is classified as AI-generated. Verified via API — no model internals needed.

**DAWN-Style Key-Derived Trigger**
A unique perturbation is generated per image using `secret_key + image_index` as seed, refined via gradient descent. Every probe image gets a different perturbation derived from the secret key. Without the key, the trigger space is astronomically large — systematic discovery is computationally infeasible.

### Layer 3 — Activation Fingerprinting

Records layer4 activation vectors (512-dim) across 200 probe images after training. A stolen model produces near-identical activations (cosine similarity > 0.85). An independently trained model scores below 0.5.

### Cryptographic Certificate

An RSA-signed JSON document containing:
- Owner ID + UTC timestamp
- SHA256 hash of watermarked model weights
- SHA256 hash of trigger pattern
- SHA256 hash of fingerprint probes

Defeats the **ambiguity attack** — if a thief adds their own watermark, the model weights change, the SHA256 hash no longer matches the certificate, and their claim collapses.

---

## What Survives What

| Attack | LSB | Spread-Spectrum | Fixed Trigger | DAWN Trigger | Fingerprint | Certificate |
|--------|-----|----------------|---------------|--------------|-------------|-------------|
| Fine-tuning (mild) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Fine-tuning (aggressive) | ⚠️ | ✅ | ⚠️ | ✅ | ⚠️ | ✅ |
| Quantization (INT8) | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trigger discovery | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Ambiguity attack | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ✅ |
| Performance degradation | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Results

```
Baseline accuracy          : 0.866900
Accuracy after watermarking: 0.866900
Delta                      : 0.00000000 ✓

Post fine-tune attack (5 epochs, lr=1e-4):
  LSB signature decoded    : YOUR_TEAM_NAME ✓
  Activation fingerprint   : 0.91 cosine similarity ✓
  Certificate              : VALID ✓
```

---

## Project Structure

```
ai-ownership/
├── model/
│   ├── evaluate.py              # accuracy measurement
│   ├── finetune_attack.py       # simulate theft
│   └── checkpoints/
│       ├── clean_resnet18_baseline.pt
│       ├── watermarked_model.pt
│       └── attacked_model.pt
├── watermark/
│   ├── weight_signature.py      # LSB + spread-spectrum
│   ├── backdoor_trigger.py      # fixed trigger + DAWN
│   ├── fingerprint.py           # activation fingerprinting
│   ├── certificate.py           # RSA signing + verification
│   └── verify.py                # unified verification runner
├── data/
│   ├── cifake/
│   ├── triggers/
│   └── fingerprints/
├── demo/
│   └── app.py                   # Gradio UI
├── main.py
├── config.yaml
└── requirements.txt
```

---

## Quickstart

```bash
# Install dependencies
pip install torch torchvision gradio cryptography numpy pillow tqdm pyyaml

# GPU (recommended)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
```

Set your owner ID in `config.yaml`:
```yaml
owner_id: "YOUR_TEAM_NAME"
```

Run the full pipeline:
```bash
python main.py --evaluate    # confirm baseline accuracy
python main.py --embed       # embed all watermarks
python main.py --attack      # simulate theft
python main.py --certify     # generate certificate
python main.py --verify      # prove ownership
python main.py --quantize    # bonus: quantization demo
```

Or all at once:
```bash
python main.py --all
```

Launch the Gradio demo:
```bash
python demo/app.py
```

---

## Demo Tabs

| Tab | What it shows |
|-----|--------------|
| 1 — Embed Watermark | Embeds all proofs, prints zero accuracy delta |
| 2 — Simulate Attack | Fine-tunes the stolen model for 5 epochs |
| 3 — Verify Ownership | Runs all 4 checks live, prints verdict |
| 4 — Quantization Demo | Shows LSB destroyed, spread-spectrum survives |

---

## Tech Stack

Python · PyTorch · torchvision · NumPy · Gradio · cryptography (RSA/SHA256)

---

## Dataset

[CIFAKE](https://huggingface.co/datasets/datasets/cifake) — 120,000 images (60k real from CIFAR-10, 60k AI-generated).

---

## Security Notes

- The private key never leaves your machine during verification — RSA signature verification requires only the public key
- In production, the certificate hash would be anchored to a public blockchain for independent timestamp verification
- DAWN triggers approximate the full Dynamic Watermarking framework post-hoc without retraining — a full DAWN implementation would require training-time integration for maximum robustness
