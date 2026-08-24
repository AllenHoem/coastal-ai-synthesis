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

```powershell
# [WIN]
# 1. Install the current AMD Adrenalin driver. Reboot. Do not skip the reboot.
# 2. Confirm the Vulkan runtime is present:
vulkaninfo --summary | Select-String "deviceName"
#    Expect: AMD Radeon RX 7900 XTX
```

Download the latest llama.cpp **Vulkan** release binary (`llama-*-bin-win-vulkan-x64.zip`) from the project's releases page and unzip to `C:\llama\`. No build step, no HIP SDK, no ROCm install.

**Record the build hash** — it goes into `run_id` (see `architecture.md` §11):
```powershell
# [WIN]
C:\llama\llama-server.exe --version
```

---

## Step 2 — Models

Pull Qwen3 GGUFs at Q4_K_M for the full D7 ladder into `C:\models\`:

| Model | Approx. Q4_K_M size | Notes |
|---|---|---|
| Qwen3-1.7B | ~1.1 GB | T0 candidate |
| Qwen3-4B | ~2.5 GB | T0/T1 boundary |
| Qwen3-8B | ~5.0 GB | T1 candidate |
| Qwen3-14B | ~9.0 GB | T1/T2 |
| Qwen3-30B-A3B | ~18 GB | MoE reference — tight against the 20GB cap (D11) |

```powershell
# [WIN]  record SHA256 for every file — these are part of run reproducibility
Get-FileHash C:\models\*.gguf -Algorithm SHA256 | Format-Table Hash,Path
```

---

## Step 3 — Start the server and verify the two hard requirements

PRD P0-4 requires per-token logprobs and parallel slots. **Verify both before writing a line of harness code** — the PRD explicitly says to confirm this before committing to a runtime.

```powershell
# [WIN]
C:\llama\llama-server.exe `
  --model C:\models\Qwen3-8B-Q4_K_M.gguf `
  --host 0.0.0.0 --port 8080 `
  --n-gpu-layers 999 `
  --ctx-size 32768 `
  --parallel 8 `
  --cont-batching `
  --metrics `
  --seed 12345
```

`--ctx-size` is the **total** across slots: 32768 / 8 = 4096 tokens per slot. See `architecture.md` §6.2 for the KV budget arithmetic that sets the ceiling.

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
# 24GB machine: allow 20GB to the GPU. Not persistent across reboot; re-run or add a launchd plist.
sudo sysctl iogpu.wired_limit_mb=20480

llama-server --model ~/models/Qwen3-8B-Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 --n-gpu-layers 999 \
  --ctx-size 32768 --parallel 8 --cont-batching --metrics --seed 12345
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
| Peak VRAM ≤ 20GB cap | measured, asserted (D11 / tradeoff T6) |
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

**`llama-server` starts but offloads zero layers.** Vulkan didn't find the device. Re-check `vulkaninfo --summary`; confirm the Adrenalin install completed and the machine was rebooted.

**Out of memory at high `--parallel`.** Expected — this is the KV budget, not a bug (`architecture.md` §6.2). Lower `--ctx-size`, or add `--cache-type-k q8_0 --cache-type-v q8_0`. Record which, because KV dtype is a trajectory field.

**DNS breaks in WSL after enabling mirrored mode.** Add to `/etc/wsl.conf`:
```ini
[network]
generateResolvConf = true
```
then `wsl --shutdown`.

**Mac server unreachable from WSL but reachable from Windows.** Mirrored mode routes WSL traffic through the host interface; confirm the Mac's macOS firewall allows incoming connections for `llama-server`.

**Corpus generation is inexplicably slow.** The repo is on `/mnt/c`. Move it to `~`.
