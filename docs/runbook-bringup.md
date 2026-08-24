# Bring-Up Runbook — Zero to First Token

Target: **~4 hours of hands-on time**, spread across reboots. Do the steps in order; each verifies before the next depends on it.

Notation: `[WIN]` = Windows PowerShell · `[WSL]` = Ubuntu shell in WSL2 · `[MAC]` = macOS Terminal

**Every fenced block is meant to be run as-is.** Comment lines inside them explain, they are never
something to type. File paths appear in prose, never as a bare comment above a command.

---

## Step 0 — WSL mirrored networking (do this first; it needs a restart)

This writes `.wslconfig` in your Windows user profile, then restarts WSL. It makes `localhost` mean
the same thing on both sides of the boundary — no host-IP lookups, no firewall rules, no
`/etc/hosts` upkeep. It removes more friction than any other single setting in this project.

**First, confirm mirrored mode is even available.** It needs WSL 2.0.0 or later *and* Windows 11
22H2 or later. On Windows 10 it does not exist. Run in PowerShell:

```powershell
wsl --version
```

If that errors, you are on the inbox WSL and the setting will be **silently ignored** no matter how
correctly you write the file — run `wsl --update` first. If you are on Windows 10, skip to the NAT
fallback below.

**Check whether you already have a config** (the next command overwrites it):

```powershell
Get-Content "$env:USERPROFILE\.wslconfig" -ErrorAction SilentlyContinue
```

**Write it.** One line, no escaping, safe to paste:

```powershell
Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value '[wsl2]','networkingMode=mirrored'
```

`Set-Content` writes each array element on its own line. The single quotes matter: unquoted,
PowerShell reads `[wsl2]` as a *type literal* and fails with `Unable to find type [wsl2]`.

> **Do not substitute a here-string** (`@"` … `"@`). It is correct PowerShell but fragile when
> pasted — if the opening `@"` does not register as a continuation, `[wsl2]` executes as its own
> statement and you get the type-literal error above.
>
> **Do not substitute `Out-File` or `>`.** Windows PowerShell 5.1 writes those as UTF-16LE, which
> WSL cannot parse — it ignores the whole file and gives no error. `Set-Content` writes ANSI, which
> is fine.

**Verify the file, then restart WSL:**

```powershell
Get-Content "$env:USERPROFILE\.wslconfig"
wsl --shutdown
```

You want exactly two lines: `[wsl2]` and `networkingMode=mirrored`.

**Confirm it took effect** — under mirrored mode WSL sees the Windows host's own interfaces, so the
host IP appears inside WSL:

```bash
# [WSL]
ip -4 addr show | grep inet
```

**Optional resource caps.** Not required, and not part of the networking fix. Add them only if WSL
is actually starving the host, and size them to your machine — `memory` above roughly half your RAM
will cause swapping, and `processors` above your logical core count is ignored:

```powershell
Set-Content -Path "$env:USERPROFILE\.wslconfig" -Value '[wsl2]','networkingMode=mirrored','memory=16GB','processors=8'
```

**If mirrored mode is unavailable** (Windows 10, or it breaks your VPN): fall back to NAT mode and resolve the host each boot —
```bash
# [WSL]  add to ~/.bashrc
export LLAMA_HOST=$(ip route show default | awk '{print $3}')
```
and add an inbound firewall rule on Windows for TCP 8080.

---

## Step 1 — GPU driver and llama.cpp on Windows

Install the current AMD Adrenalin driver and reboot — do not skip the reboot. Then confirm the
Vulkan runtime sees the card:

```powershell
vulkaninfo --summary | Select-String "deviceName"
```

Expect **AMD Radeon RX 7900 XT** — 20GB, 320-bit, ~800 GB/s, 315W board power.

### You have two Vulkan devices — pin the right one

A Ryzen desktop CPU carries integrated graphics, so `vulkaninfo` reports both:

```
GPU0 … deviceName = AMD Radeon RX 7900 XT            ← discrete, 20GB, the one you want
GPU1 … deviceName = AMD Radeon(TM) Graphics          ← CPU integrated graphics
```

**llama.cpp enumerates both.** If it selects the iGPU you get either an immediate allocation
failure or throughput an order of magnitude off, with nothing in the log saying why. Confirm the
index and pin it explicitly:

```powershell
C:\llama\llama-server.exe --list-devices
```

Note which entry is the 7900 XT — usually `Vulkan0` — and pass `--device Vulkan0` on every launch.
On builds without `--device`, set `GGML_VK_VISIBLE_DEVICES=0` in the environment instead.

### Move the display to the iGPU

Optional for the small models, **required for 30B-A3B**. With a monitor attached to the 7900 XT,
Windows reserves roughly 1.5GB of its 20GB. Plugging the monitor into the motherboard instead
hands that back, and the iGPU drives a desktop without difficulty.

| Display attached to | Usable for model + KV |
|---|---|
| 7900 XT | ~18.5 GB |
| Motherboard (iGPU) | ~19.5 GB |

That 1GB is the difference between the MoE reference model loading and not (Step 2).

Download the latest llama.cpp **Vulkan** release binary (`llama-*-bin-win-vulkan-x64.zip`) from the project's releases page and unzip to `C:\llama\`. No build step, no HIP SDK, no ROCm install.

**Record the build hash** — it goes into `run_id` (see `architecture.md` §11):
```powershell
# [WIN]
C:\llama\llama-server.exe --version
```

---

## Step 2 — Models

Pull Qwen3 GGUFs at Q4_K_M for the full D7 ladder into `C:\models\`. All five are the same
family, tokenizer, and quantization, so model size is the only variable (D6/D7/D8).

| Model | Q4_K_M weights | KV per token (f16) | Fit on 20GB |
|---|---|---|---|
| Qwen3-1.7B | ~1.1 GB | 112 KiB | Comfortable |
| Qwen3-4B | ~2.5 GB | 144 KiB | Comfortable |
| Qwen3-8B | ~5.0 GB | 144 KiB | Comfortable |
| Qwen3-14B | ~9.0 GB | 160 KiB | Fits; limits concurrency |
| Qwen3-30B-A3B | ~18.6 GB | 96 KiB | **Only with the display on the iGPU** |

`Qwen3-30B-A3B` at Q4_K_M leaves under 1GB for KV cache even with the display moved off the card.
That is enough for two to four slots at short context and nothing more. **This is fine** — D12 puts
it in the sweep as a MoE bandwidth reference so the projection math transfers to T3, not as an
accuracy-sweep participant. Run it at concurrency 1–2, record the throughput, and let the smaller
models carry the accuracy work.

Do **not** drop it to Q4_K_S or IQ4_XS to make it fit. D8 fixes one quantization across the ladder;
changing it for one rung turns a size comparison into a size-and-quant comparison.

```powershell
Get-FileHash C:\models\*.gguf -Algorithm SHA256 | Format-Table Hash,Path
```

Record every hash — they are part of `run_id` (`architecture.md` §11).

---

## Step 3 — Start the server and verify the two hard requirements

PRD P0-4 requires per-token logprobs and parallel slots. **Verify both before writing a line of harness code** — the PRD explicitly says to confirm this before committing to a runtime.

```powershell
C:\llama\llama-server.exe `
  --model C:\models\Qwen3-8B-Q4_K_M.gguf `
  --device Vulkan0 `
  --host 0.0.0.0 --port 8080 `
  --n-gpu-layers 999 `
  --ctx-size 65536 `
  --parallel 16 `
  --cont-batching `
  --flash-attn `
  --metrics `
  --seed 12345
```

`--ctx-size` is the **total** across slots: 65536 / 16 = 4096 tokens per slot, which comfortably
holds an invoice plus the prompt. `--device Vulkan0` is not optional — see Step 1.

### Slot budgets for a 20GB card

Concurrency is bounded by KV cache, not by the scheduler:

```
slots_max = (usable_vram − weights) / (ctx_per_slot × kv_bytes_per_token)
kv_bytes_per_token = 2 × n_layers × n_kv_heads × head_dim × bytes_per_element
```

Computed for this card at 4096 tokens per slot, display on the dGPU (~18.5GB usable). Verify the
layer and head counts against the metadata `llama-server` prints at load:

| Model | `--parallel` (f16 KV) | `--ctx-size` | `--parallel` (q8 KV) | `--ctx-size` |
|---|---|---|---|---|
| Qwen3-1.7B | 32 | 131072 | 128 *(at 2048/slot)* | 262144 |
| Qwen3-4B | 24 | 98304 | 56 | 229376 |
| Qwen3-8B | 16 | 65536 | 48 | 196608 |
| Qwen3-14B | 8 | 32768 | 30 | 122880 |
| Qwen3-30B-A3B | 2 *(iGPU display)* | 8192 | 4 | 16384 |

Add `--cache-type-k q8_0 --cache-type-v q8_0` for the q8 columns. **Record which you used** — KV
dtype is a trajectory field, and it is an accuracy confound, which is why Pass A pins f16 and only
Pass B quantizes (ADR-0009).

**The P0-9 concurrency axis does not fully fit this card, and that is a result rather than a
problem.** At 4096 tokens per slot with q8 KV, batched concurrency tops out around 32 for the
1.7B–8B models and around 8 for 14B. Offer 128 anyway: the extra requests queue instead of
batching, aggregate throughput still rises, and per-request latency grows linearly. Both numbers go
in every record — `offered_concurrency` and `server_slots` — precisely so a queued-128 result is
never reported as a batched-128 result (`architecture.md` §6.2).

**Verification A — logprobs are actually returned:**
```bash
# [WSL]
curl -s localhost:8080/v1/chat/completions -H 'Content-Type: application/json' -d '{
  "model":"local","messages":[{"role":"user","content":"Reply with the single word: ok"}],
  "max_tokens":5,"temperature":0,"logprobs":true,"top_logprobs":5
}' | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d['choices'][0].get('logprobs'), indent=2)[:400])"
```
A populated `content[].logprob` array means P0-4 is satisfied. **A null here stops the project** — flag it and re-evaluate the runtime before proceeding.

**Verification B — slots batch rather than queue:**
```bash
# [WSL]
time (for i in $(seq 1 8); do
  curl -s localhost:8080/v1/chat/completions -H 'Content-Type: application/json' \
    -d '{"model":"local","messages":[{"role":"user","content":"Count to twenty."}],"max_tokens":120,"temperature":0}' >/dev/null &
done; wait)
```
Eight concurrent requests should take meaningfully less than eight sequential ones. If wall-clock scales linearly with request count, `--parallel` is not engaging — check that `--cont-batching` is set.

**Verification C — determinism:**
Run the same `temperature: 0` prompt three times. Byte-identical completions are required for reproducibility. Any drift at c=1 is a red flag worth resolving now, not during a sweep.

---

## Step 4 — WSL environment

```bash
# [WSL]  keep the repo on ext4, NOT /mnt/c  (see tradeoffs T5)
cd ~ && git clone <repo> coastal-ai-synthesis && cd coastal-ai-synthesis

curl -LsSf https://astral.sh/uv/install.sh | sh
uv sync

# WeasyPrint needs a few system libraries; this is the only apt step in the project
sudo apt-get update && sudo apt-get install -y \
  libpango-1.0-0 libpangoft2-1.0-0 libharfbuzz0b libffi-dev

cp .env.example .env    # then add ANTHROPIC_API_KEY and node addresses
```

**Verify the boundary:**
```bash
# [WSL]
curl -s localhost:8080/health   # mirrored mode
# or
curl -s $LLAMA_HOST:8080/health # NAT fallback
```

---

## Step 5 — MacBook Pro M4 as the T1 anchor

```bash
# [MAC]
brew install llama.cpp

# Raise the GPU wired-memory limit — the default reserves too much for the host.
# 24GB machine: allow 20GB to the GPU — matches the 7900 XT's usable budget, so the two
# nodes run identical --parallel/--ctx-size configs. Not persistent across reboot.
sudo sysctl iogpu.wired_limit_mb=20480

llama-server --model ~/models/Qwen3-8B-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --n-gpu-layers 999 \
  --ctx-size 65536 --parallel 16 --cont-batching --flash-attn --metrics --seed 12345
```

**Give both machines DHCP reservations in the router.** Five minutes now; otherwise `.env` needs editing whenever a lease rotates (constraint C4).

```bash
# [WSL]  verify LAN reachability
curl -s http://<macbook-reserved-ip>:8080/health
```

**Free power metering on this node** — this is the project's only measured-energy anchor (tradeoff T2):
```bash
# [MAC]
sudo powermetrics --samplers cpu_power,gpu_power -i 1000 -n 5 | grep -E "Combined|GPU Power"
```

**Characterize it before trusting the tier label:**
```bash
# [WSL]
make probe-node NODE=mac
```
An M4 *base* is ~120 GB/s and lands on T1. An M4 *Pro* is ~273 GB/s — T3 bandwidth with T1 capacity, which is a different (and interesting) story. The projection model fits from measurement either way, but the narrative depends on which it is.

---

## Step 6 — Readiness gate

```bash
# [WSL]
make bringup
```

Prints a table and refuses to pass until every row is green:

| Check | Passing looks like |
|---|---|
| Windows node reachable | `/health` 200 from WSL |
| Logprobs returned | non-null `content[].logprob` |
| Parallel slots engaging | 8 concurrent < 3x single-request wall-clock |
| Determinism at temp 0 | three identical completions |
| Correct GPU selected | `--list-devices` output pinned; not the iGPU |
| Peak VRAM ≤ 20GB | measured, asserted — D11's assumption is now the physical limit |
| Mac node reachable | `/health` 200 over LAN |
| Mac `powermetrics` available | sampler returns watts |
| Anthropic key valid | 1-token probe, `usage` returned |
| Rate card current | `effective_until` not in the past |
| Repo on ext4 | not under `/mnt/c` |
| Disk free | ≥ 40GB for models + corpus + runs |

---

## Troubleshooting

**`Unable to find type [wsl2]` / `invalid argument: ([wsl2])` when writing `.wslconfig`.** The
here-string broke apart and PowerShell evaluated `[wsl2]` as a type literal. Use the single-line
`Set-Content -Value '[wsl2]','networkingMode=mirrored'` form in Step 0, or edit the file in
Notepad. Also check you gave `Set-Content` a `-Path` — with none, it has no destination.

**`.wslconfig` looks right but WSL ignores it.** Two usual causes. Either the file is UTF-16
(written with `Out-File` or `>` under Windows PowerShell 5.1) — rewrite it with `Set-Content`;
or `wsl --version` errors, meaning you are on the inbox WSL, which predates mirrored networking
and discards the setting without complaint. `wsl --update`, then `wsl --shutdown`.

**`llama-server` starts but offloads zero layers.** Vulkan didn't find the device. Re-check
`vulkaninfo --summary`; confirm the Adrenalin install completed and the machine was rebooted.

**Throughput is an order of magnitude below the table, or it OOMs on a model that should fit.**
llama.cpp selected the integrated GPU. Run `--list-devices` and pass `--device Vulkan0` explicitly
(Step 1). This fails quietly — nothing in the log announces which device was chosen.

**`Qwen3-30B-A3B` fails to allocate.** Expected with the display on the 7900 XT: weights alone are
~18.6GB of ~18.5GB usable. Move the monitor to the motherboard's output, or skip this rung — it is
a MoE bandwidth reference (D12), not an accuracy-sweep participant.

**Out of memory at high `--parallel`.** Expected — this is the KV budget, not a bug (`architecture.md` §6.2). Lower `--ctx-size`, or add `--cache-type-k q8_0 --cache-type-v q8_0`. Record which, because KV dtype is a trajectory field.

**DNS breaks in WSL after enabling mirrored mode.** Add to `/etc/wsl.conf`:
```ini
[network]
generateResolvConf = true
```
then `wsl --shutdown`.

**Mac server unreachable from WSL but reachable from Windows.** Mirrored mode routes WSL traffic through the host interface; confirm the Mac's macOS firewall allows incoming connections for `llama-server`.

**Corpus generation is inexplicably slow.** The repo is on `/mnt/c`. Move it to `~`.
