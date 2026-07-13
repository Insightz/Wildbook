# Runbook — Wildbook + WBIA on Windows (WSL2 + Docker Desktop), for carp

Target machine: Windows, 32 GB RAM, NVIDIA RTX 4070 laptop (8 GB VRAM).
Goal: run the whole stack locally and try venue-scoped carp identification.

> **Do everything inside WSL2 Ubuntu.** The build scripts (`frontend/maven-build.sh`,
> `docker-entrypoint.sh`, `wildbook-ia/devops/build.sh`) are bash, and Windows
> line endings (CRLF) + native paths break them. WSL2 also gives Docker
> Desktop access to your GPU.

---

## 0. One-time Windows setup

1. Install **Docker Desktop** and enable the **WSL2 backend**
   (Settings → General → *Use the WSL 2 based engine*).
2. Install WSL2 + Ubuntu:  in PowerShell → `wsl --install -d Ubuntu`
3. Install the current **NVIDIA driver** for the 4070 on Windows (Game Ready or
   Studio). That driver is what exposes CUDA to WSL2 — you do **not** install a
   CUDA toolkit on Windows.
4. Docker Desktop → Settings → **Resources**: give it ~20 GB RAM (of your 32),
   and under **WSL Integration** enable your Ubuntu distro.

---

## 1. One-time WSL2 (Ubuntu) setup

Open the Ubuntu terminal and run:

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk maven git curl
# Node >= 18 (NodeSource gives a current version):
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

java -version && mvn -version && node -v && docker version   # sanity check
```

**OpenSearch kernel setting.** Docker Desktop runs containers in its own
`docker-desktop` WSL distro, so set `vm.max_map_count` there (not in Ubuntu):

```powershell
# from Windows PowerShell:
wsl -d docker-desktop -u root sysctl -w vm.max_map_count=262144
```

To make it survive reboots, create `%UserProfile%\.wslconfig` with:

```
[wsl2]
kernelCommandLine = sysctl.vm.max_map_count=262144
```

then `wsl --shutdown` and restart Docker Desktop.

---

## 2. Clone the repos (inside the WSL2 home dir, NOT /mnt/c)

Working under `/mnt/c/...` is slow for Docker bind mounts and reintroduces CRLF
problems. Use the Linux home:

```bash
cd ~
git clone https://github.com/Insightz/Wildbook.git
git clone https://github.com/Insightz/wildbook-ia.git      # only needed for Rung 3
```

---

## 3. Build Wildbook (the .war — includes the React frontend)

```bash
cd ~/Wildbook

# The template already contains everything the carp compose needs (WBIA_DB_URI
# is in there). Its defaults are fine for local dev. No edits required.
cp devops/development/_env.template devops/development/.env

printf 'PUBLIC_URL=/react/\nSITE_NAME=My Local Wildbook\n' > frontend/.env

npm install
(cd frontend && npm install)

mvn clean install          # builds React (via maven-build.sh) + the .war
ls target/wildbook-*.war   # confirm it exists
```

Deploy the war into the directory the container mounts:

```bash
mkdir -p ~/wildbook-dev/webapps/wildbook ~/wildbook-dev/logs
cd ~/wildbook-dev/webapps/wildbook
jar -xvf ~/Wildbook/target/wildbook-*.war
```

(`WILDBOOK_BASE_DIR` in `.env` defaults to `~/wildbook-dev` — keep it or change
both.)

---

## 4. Start the full stack (db + wildbook + opensearch + smtp + wbia)

Copy `docker-compose.carp.yml` (the reconciled file) into
`~/Wildbook/devops/development/`, then:

```bash
cd ~/Wildbook/devops/development
docker compose -f docker-compose.carp.yml up
```

First run pulls images (the WBIA image is large) and Tomcat unpacks the war —
give it a few minutes. Verify all five services are healthy:

```bash
docker compose -f docker-compose.carp.yml ps
```

Open **http://localhost:81/** → log in **tomcat / tomcat123**.

Rebuild loop after code changes: `mvn clean install`, re-extract the war (step
3), then `docker compose -f docker-compose.carp.yml restart wildbook`.

---

## 5. First experiment — venue-scoped ID with NO trained model (Rung 2)

HotSpotter is a generic feature matcher, so you can prove the core idea before
training anything:

1. In the UI, make sure `carp` is a selectable species and you have at least one
   Location (venue). (These come from `commonConfiguration.properties` /
   `IA.json` / `locationID.json` — ask me for the pre-filled carp versions.)
2. Add several encounters at **one location**, each with a carp photo; draw an
   annotation (box + species=carp); assign each to a named individual. This is
   your "stock."
3. Add one more encounter at that location (the "catch"); annotate it.
4. Run identification, choosing **HotSpotter**, scoped to that location.
5. You should get a ranked candidate list drawn only from that venue's fish.

That validates the whole "given a catch, return the best matches from this venue
only" flow with zero model training.

---

## 6. Rung 3 — train carp models (optional, later)

Only needed to auto-assign species/boxes (so you are not annotating by hand).

1. Build a WBIA image from your fork (has the carp scaffolding already committed):
   ```bash
   cd ~/wildbook-ia/devops
   ./build.sh wbia-base wbia-provision wbia
   ```
2. Point the `wbia` service `image:` in `docker-compose.carp.yml` at your tag.
3. Enable the GPU block in that service (see the comment there) after confirming
   `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi`
   lists the 4070.
4. Train the labeler (`ibs.labeler_train(species_list=['carp'])`) — see
   `wildbook-ia/docs/training_custom_species.md`.

**VRAM note:** the labeler's default `batch_size=48` can OOM on 8 GB. Pass a
smaller batch (≈16–24) when you get there.

---

## Gotchas

- **Everything in WSL2 home (`~`), not `/mnt/c`** — speed + line endings.
- The public `wildme/wbia` image may be **CPU-only**; if the host `nvidia-smi`
  works but the container doesn't see the GPU, that's why. HotSpotter still runs
  fine on CPU; build from your fork for GPU training.
- If OpenSearch container restarts on boot: the `vm.max_map_count` setting did
  not persist — re-run the step-1 command.
- The carp path uses **WBIA**, which Wild Me has flagged as deprecated in favor
  of a newer ml-service. It works and is what your scaffolding targets; just
  know the upstream direction.
