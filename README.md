# Advanced-Guide-Validator-and-GPU-Miner-Set-up-on-FLOP
This guide provides a step-by-step walkthrough for setting up a **FLOP Network Validator** (CPU) and a **FLOP GPU Miner** (inference backend + miner worker) on FLOP. It covers everything from hardware selection and environment preparation and troubleshooting. This guide follows the official operator docs at [docs.flop.network/operators](https://docs.flop.network/operators), with corrections for missing prerequisites.

<img width="986" height="492" alt="image" src="https://github.com/user-attachments/assets/46cc9d35-1d07-46d2-943b-5c2af449dbc3" />


----
# PLEASE TAKE NOTE! ! !
## THE FLOP DEVNET IS DISPOSABLE AND EARNS NOTHING. NO REWARDS, NO AIRDROP WEIGHT!
### RUN IT ONLY FOR THE TECH, OR TO BE READY FOR TESTNET. PASSING THAT INFO!

---

- **Public operator enrollment is NOT open yet.** Validators need approval and funding from FLOP operations. **Validator package `validator-cs-be91fe83 · 0.3.0-devnet.1` is live**
- Devnet onboarding stays **administrator-assisted**; self-service public testnet onboarding is still closed. Miners are **invite-only**, and there's no public intake contact yet.
- **There is NO FLOP token.** yet. 

----
### The setup covers two separate roles. It includes:
  * Hardware requirements for each role.
  * The validator onboarding package flow (`pnpm onboard prepare → join → verify`).
  * The GPU miner's flop CLI, keystore, sponsorship, model fetch, `.env` template and Docker Compose run.
  * Useful commands and troubleshooting tables for both roles.
  * Migrating your validator (or miner) to a new server without changing keys or fingerprint.
### By following this guide you'll understand the full operator flow and be ready to move fast when testnet onboarding opens.


------
> **⚠️ Important Notes**
>
> - **NEVER share your seed phrase, keystore password or session token** with anyone, whether it's "support", a DM, or an AI agent. FLOP never asks for your seed. Never pay a deposit to claim a token, mint or airdrop; protocol staking is separate.
> - **Do NOT use the same account for a Validator and a Miner.** Create a dedicated sr25519 miner account.
> - **Never run two validators with the same session keys.** Simultaneous voting = equivocation.
> - Only trust links on the [official links page](https://docs.flop.network/docs/official-links).

------
## 1. Preparation and Requirements
- [ ] Read the [FLOP Operator Docs](https://docs.flop.network/operators)
- [ ] Choose your role:

| Role | What you run | GPU? | Success looks like |
|---|---|---|---|
| **Validator** | CPU node (BABE blocks + AlephBFT finality) + DA worker | No | Active session/BABE/Aleph keys, advancing finality |
| **GPU Miner** | `miner-vllm-toploc` GPU backend + `miner-worker` | Yes (NVIDIA) | Measured inference → registration → accepted calibration; accepted paid work is a later gate |

- [ ] **Validator:** get approved by FLOP operations
- [ ] **Miner:** get an invite from the devnet operator (they give you images, Compose files, `MODEL_HASH` and a `DA_URL`)
- [ ] Install **Docker** and **Docker Compose v2**


## 2. Public Devnet Endpoints
| Setting | Value |
|---|---|
| RPC | `wss://rpc.devnet.flop.network` |
| Ghost mailbox | `https://mailbox.devnet.flop.network` |
| Faucet / stake sponsor | `https://faucet.flop.network` |
| Explorer | `https://explorer.devnet.flop.network` |
| Releases manifest | `https://flop.network/releases.json` |

---

## 🖥 Recommended Server Specs

---
### ⚙️ Validator Node
- [ ] **CPU:** 8 dedicated cores
- [ ] **RAM:** 32 GB
- [ ] **Storage:** 4 TB NVMe (DA custody is why storage is so large.)
- [ ] **Network:** reliable, low-latency 1 Gbps

### ⚙️ GPU Miner
- [ ] **GPU:** NVIDIA. The **A40** is the validated card (Qwen2.5-0.5B-Instruct, BF16)
- [ ] **Host:** a full VM or owned Linux host (**Ubuntu 24.04** for the installer pack) with Docker, Compose, NVIDIA driver and Container Toolkit
- [ ] **Storage:** enough for the checkpoint plus persistent volumes (about 60 GB for Qwen2.5-0.5B on RunPod)
- [ ] **Network:** a public DNS name with ports **80/443** open (or your own tunnel to `127.0.0.1:8645`)
      
**Why these specs:**
- Container-only rentals (RunPod Pods, GPU.ai containers) **can't** run the Compose pair.
- A faster GPU means a larger capacity-stake requirement.

> **Note:**
> Stopping Compose **does NOT stop VM billing**. Delete the instance in your provider console when you're done.

-------

## 🏅 Devnet Miner Limits
- [ ] At most **10** new sponsored miners per rolling 24h
- [ ] At most **20M** sponsored (test) FLOP per 24h. A new miner starts at about **1M**, and top-ups go to at most **2M**
- [ ] **One** sponsorship per account
- [ ] Sponsor route: **6 requests/min** per address
> **Tip:** **403** = not approved yet. **429** = you hit a daily limit, retry tomorrow. **503** = retry later.


------
## Getting Started

Follow the **manual** validator and GPU miner steps below. An optional interactive autorun menu sits at the end of this guide if you prefer a menu-driven path.

##  Step 1: Set Up the VALIDATOR

This server runs **two services**:
- **Validator node** → authors blocks (BABE) and votes on finality (AlephBFT)
- **DA worker** → serves assigned data
---

### 1️⃣ Install Essentials
You need Docker Engine, the Compose plugin, `curl`, `jq`, **Node.js 24**, **pnpm 12.14.0** and `age`.

```
sudo apt update && sudo apt upgrade -y && \
sudo apt install -y curl jq ca-certificates gnupg lsb-release
```

### Add Docker’s official GPG key
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --yes --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Set up repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

<img width="1876" height="480" alt="image" src="https://github.com/user-attachments/assets/30c716cf-4f3e-4a75-90e0-e7ba751b2dc1" />


### Install Docker Engine & Compose (Ubuntu only)
```
sudo apt update && \
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify installation
```
docker --version
docker compose version
docker info
```
<img width="1884" height="366" alt="image" src="https://github.com/user-attachments/assets/df3e0eae-5836-4330-8ecc-eb44864870d3" />

> If `docker info` returns a socket permission error, run as `root` or deliberately configure Docker access for your non-root user. Docker-group membership effectively grants root-level control of the host.

### Install Node 24, pnpm 12.14.0 and age

```
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs

sudo corepack enable
sudo corepack prepare pnpm@12.14.0 --activate
sudo apt-get install -y age

node -v    # expect v24.x
pnpm -v    # expect 12.14.0
age --version
```
<img width="1876" height="336" alt="image" src="https://github.com/user-attachments/assets/602674ef-24f3-4198-bdd4-6c8a1b75bee8" />

------

## 2️⃣ 📥 Download the Validator Package
> ✅ **STATUS (6 Nov 2026): LIVE.** Package listing: `validator-cs-be91fe83 · 0.3.0-devnet.1 · FLOP Devnet · Linux amd64`. Devnet onboarding is still **administrator-assisted**; self-service public testnet onboarding is **not open yet**. Track access: [flop-labs/network#10](https://github.com/flop-labs/network/issues/10).

### Download, verify and extract
```
mkdir -p ~/flop && cd ~/flop
wget https://explorer.devnet.flop.network/validator-downloads/validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
wget https://explorer.devnet.flop.network/validator-downloads/validator-cs-be91fe83-0.3.0-devnet.1.tar.gz.sha256
sha256sum -c validator-cs-be91fe83-0.3.0-devnet.1.tar.gz.sha256
tar -xzf validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
cd validator-cs-be91fe83 && cat README.md
```

<img width="1880" height="678" alt="image" src="https://github.com/user-attachments/assets/21b1fef9-b983-4b1a-a56e-bc11c2f08093" />

- `OK` means the file is genuine and safe to use. `FAILED` means delete it and download it again.


## 3️⃣ Prepare & Send Your Public Values
---
### Set hostname (do this before prepare)
`prepare` does **not** ask for a name — it uses the host's current hostname (often a provider default like `165093.example.us`). Ops enrolls Tailscale under that exact name, so set something stable **first** if you want a cleaner label: 

### CHANGE *XXXXXX* TO YOUR NAME
```
sudo hostnamectl set-hostname xxxxxx
# open a new shell (or reconnect SSH), then confirm:
hostname -f 2>/dev/null || hostname
```
### Do not rename the host after ops has enrolled you.


### Install dependencies and prepare

```
node preflight.mjs
pnpm install --frozen-lockfile --ignore-scripts
pnpm onboard prepare
```
<img width="1814" height="356" alt="image" src="https://github.com/user-attachments/assets/8f841176-c32a-45ff-935b-03366ae77660" />


Default profile is `administrator-assisted-devnet` (you can pass `--profile administrator-assisted-devnet` explicitly).

`prepare` prints two public values and writes private files locally (`account.json`, `.env`, mode `600`). Example output:
```
Hostname: 165093.example.us
Public SS58 address: 5HKsuQanpUH8GLFEJkmdyiKDAKUnTSogHATxsht5vJxt8Psr
Send these two public values to operations. Keep account.json and .env private.
```

### Save / view what you send to operations
`prepare` does **not** print the host SSH key you will need to grab that separately. 

```
mkdir -p ~/flop
PUBLIC_NOTE=~/flop/ops-public-values.txt

{
  echo "Hostname: $(hostname -f 2>/dev/null || hostname)"
  echo "Public SS58 address: $(jq -r '.address // .ss58Address // .publicAddress // empty' account.json 2>/dev/null || true)"
  echo "Host SSH public key:"
  cat /etc/ssh/ssh_host_ed25519_key.pub
  echo
  echo "Host SSH fingerprint (confirm on a second channel):"
  ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
} | tee "$PUBLIC_NOTE"

chmod 644 "$PUBLIC_NOTE"
cat "$PUBLIC_NOTE"
```


### Who to send this to
Use the **FLOP team member already coordinating your invitation** (DM / email / chat you already have with them). That is the only place these public values go.

- If you **do not** have a named ops contact yet, request an introduction in [flop-labs/network#10](https://github.com/flop-labs/network/issues/10).
- Posting on that issue does **not** grant admission by itself.
- Do **not** post seeds, `account.json`, `.env`, keystores or private keys anywhere public.

### What to send (public only)
Send **only** these three public values (from `$PUBLIC_NOTE` above):
1. **Hostname** (from `prepare` / `hostname`)
2. **Public SS58 address** (from `prepare`)
3. **Host SSH public key** (`cat /etc/ssh/ssh_host_ed25519_key.pub`) — ops encrypts Tailscale enrollment to this key

Confirm the `SHA256:` fingerprint from `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` on a **second channel** (call / other chat).

**Never send:** seed, `account.json`, `.env`, node keystore, or any private SSH key. Back those up privately on the host only.

### Ports
| Port | Use | Exposure |
|---|---|---|
| `30333` TCP | P2P | tailnet |
| `7001` TCP | DA | tailnet |
| `9944` | RPC | **loopback only** |
| `9615` | metrics | **loopback only** |

## 4️⃣ Wait for ops, then Join
**Do not run `pnpm onboard join` or `pnpm onboard verify` yet.** After prepare there is no local node on `ws://127.0.0.1:9944`, so verify will just loop on `API-WS: disconnected … 1006`.

Wait until operations has given you **all** of the following:
1. **Tailscale enrollment** — encrypted key / `tailscale up` for your hostname (save as `tailnet-key.age`, decrypt with the host SSH key, enroll; leave Tailscale SSH off: `sudo tailscale set --ssh=false`)
2. **Funding** — your public SS58 funded above the live stake floor
3. **`da-peers.json`** — approved private DA peer directory; save it beside the package README (join refuses a missing, empty or non-tailnet file)
4. **Access-check confirmation** — ops OK that unintended services are blocked and you can reach P2P/DA peers

Only then:
```
pnpm onboard join
```
Then tell operations **"ready"**. This locks stake.
> ⚠️ If a submission fails or only partly goes through, **STOP** and contact operations. Do not blindly rerun signed actions.


## 5️⃣ Verify (only after operations activates you)
```
pnpm onboard verify
```
"Ready" doesn't mean "active". You need babe/audi/alp0 session keys plus advancing finality. Run this **after** ops confirms activation — not right after prepare.




### Migrating the Validator to a New Server
Use this when moving your validator to another VPS (better specs, cheaper host, etc.). The goal is to carry over the **same keys, config and host SSH key**, so your SS58 address and `SHA256:` fingerprint stay identical.

> ⚠️ **Read first**
> - **Never run the validator on both servers at once** with the same keys. Old server stops first, and stays stopped.
> - If ops is already enrolling or has enrolled you, **tell your ops contact before moving**. The IP changes, and they verify the host key on their side.
> - The backup contains **private keys**. Never upload it anywhere, never paste it in chat, delete it after the move.

### What gets copied
| Item | Path | Why |
|---|---|---|
| Validator package, `account.json`, `.env`, keystore, data | `~/flop` (e.g. `/root/flop`) | Your validator identity and config |
| Host SSH key pair | `/etc/ssh/ssh_host_ed25519_key` + `.pub` | Keeps the same `SHA256:` fingerprint ops confirmed |

**Not copied:** Docker, Node, pnpm, `age`, Tailscale, firewall rules. Reinstall those on the new server (Section 1️⃣).

### Step 1: On the OLD server, stop and bundle
```
cd ~/flop/validator-cs-be91fe83 && docker compose down 2>/dev/null
tar czf /root/validator-backup.tgz \
  /root/flop \
  /etc/ssh/ssh_host_ed25519_key /etc/ssh/ssh_host_ed25519_key.pub
sha256sum /root/validator-backup.tgz
```

### Step 2: On the NEW server, pull it over SSH
Run these as **separate lines** (not on one line):
```
scp root@OLD_SERVER_IP:/root/validator-backup.tgz /root/
sha256sum /root/validator-backup.tgz
```
The hash must match the old server's exactly. Or let the server check it for you:
```
[ "$(ssh root@OLD_SERVER_IP sha256sum /root/validator-backup.tgz | cut -d' ' -f1)" = "$(sha256sum /root/validator-backup.tgz | cut -d' ' -f1)" ] && echo MATCH || echo MISMATCH
```
`MISMATCH` = delete the copy and `scp` again.

### Step 3: On the NEW server, restore
### CHANGE *XXXXXX* TO THE SAME HOSTNAME YOU USED BEFORE
```
tar xzf /root/validator-backup.tgz -C /
chmod 600 /etc/ssh/ssh_host_ed25519_key
chmod 644 /etc/ssh/ssh_host_ed25519_key.pub
sudo hostnamectl set-hostname xxxxxx
systemctl restart ssh
```
> Your SSH client will warn that the host key for the new IP is already known for another host. That's expected, since you moved the key on purpose.

### Step 4: Reinstall tools and check
Install Docker, Node 24, pnpm 12.14.0 and `age` exactly as in Section 1️⃣, then:
```
hostname                                        # same name as before
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub # same SHA256: fingerprint as before
cd ~/flop/validator-cs-be91fe83
node preflight.mjs
pnpm install --frozen-lockfile --ignore-scripts
```
- **Do NOT run `pnpm onboard prepare` again.** That would create a new account. Your existing `account.json` and `.env` are already restored.

### Step 5: Tell ops, then clean up
Send your ops contact something like:
```
I've moved <hostname> to a new server at <NEW_IP>.
Same SS58: <your SS58>
Same host-key fingerprint: SHA256:<...>
Old server is stopped.
```
Then:
```
rm -f /root/validator-backup.tgz        # on BOTH servers
```
And wipe `~/flop` on the old server or destroy the old instance.

After that, carry on from Section 4️⃣ (wait for ops → join → verify) as normal.

> 💡 **Moving the GPU miner too?** Same idea. Bundle `/opt/flop-miner` (includes `miner.env`, `signer`, `session-keys`) instead of `~/flop`. Let in-flight sessions finish first, and never run the same miner key on two machines.


-----------

----
##  More Info & Resources

- **FLOP Operator Docs:** [https://docs.flop.network/operators](https://docs.flop.network/operators)
- **Official Links (verify everything here):** [https://docs.flop.network/docs/official-links](https://docs.flop.network/docs/official-links)
- **Docs Assistant:** [https://ask.flop.finance](https://ask.flop.finance)
- **Bug Reports:** [https://github.com/flop-labs/network](https://github.com/flop-labs/network)
- **Devnet Explorer:** [https://explorer.devnet.flop.network](https://explorer.devnet.flop.network)
- **FLOP on X:** [https://x.com/flop_labs](https://x.com/flop_labs)
- **My Twitter:** [https://twitter.com/mztacat](https://twitter.com/mztacat)
- **My Discord:** `mztacat_`
