**Tech Stack**

| Layer | Tool | Purpose |
|---|---|---|
| Primitive sizes | Python `cryptography` lib | Measure actual ECC key/cert DER sizes |
| PQ crypto | `liboqs` + `liboqs-python` v0.14.1 | ML-KEM-768 and ML-DSA-65 at runtime |
| Simulation | Python 3.11 (pure, no framework) | Model message flows per phase |
| Output | `tabulate` + `matplotlib` | Paper-ready tables and bar charts |
| Memory profiling | `psutil` + stdlib `tracemalloc` | Per-session RSS and heap allocation |
| Sandboxing | Docker `linux/arm/v7`, 512 MB RAM, 1 CPU | eUICC-class constrained device emulation |

---

**Ubuntu-specific recommendations**

The benchmark is designed to run inside a Docker container, so the host OS only matters for build speed and driver availability. Ubuntu 22.04 LTS or 24.04 LTS is recommended over macOS for the following reasons:

*liboqs install (no source build required on Ubuntu):*
```bash
sudo add-apt-repository ppa:openquantumsafe/liboqs
sudo apt install -y liboqs-dev
pip install liboqs-python
```
On macOS, liboqs must be compiled from source via Homebrew or cmake, adding ~10 minutes to setup. The OQS PPA provides pre-built `.deb` packages for Ubuntu 22.04/24.04.

*Docker arm/v7 emulation is faster on Ubuntu:*
On macOS, Docker runs inside a Linux VM (Apple Hypervisor), so QEMU `linux/arm/v7` emulation runs on top of an already-virtualised layer. On Ubuntu, Docker uses native Linux cgroups and `binfmt_misc` directly -- arm/v7 image build time drops from ~40 min (macOS) to ~15--20 min (Ubuntu).

*psutil measurements are more accurate on Ubuntu:*
`psutil` reads from `/proc/self/status` and `/sys/fs/cgroup/memory.current` on Linux -- native kernel interfaces. On macOS it falls back to Mach APIs which are less granular. The provisioning simulation RSS numbers will be more reliable and reproducible on Ubuntu.

*CI/CD:*
GitHub Actions `ubuntu-24.04` runners are free for public repos and support Docker buildx with QEMU out of the box. macOS runners cost ~10x more per minute. For a reproducible paper artefact, an Ubuntu-based CI pipeline is strongly recommended.

*Recommended Ubuntu setup (one-time):*
```bash
# Register QEMU binfmt handlers for arm/v7
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Verify arm/v7 support
docker run --rm --platform linux/arm/v7 alpine uname -m  # should print armv7l

# Build and run the benchmark container
cd benchmark/
docker compose up --build
```

---

**Metrics to collect per variant**

For each of the 4 phases (InitiateAuthentication, AuthenticateClient, GetBoundProfilePackage, InstallConfirmation):
- Total message size (bytes)
- Cumulative bandwidth across the full handshake
- Individual key and cert sizes as a separate breakdown table

---

**Primitive mapping per variant**

| Primitive | Classical | PQ-RSP | KEMRSP |
|---|---|---|---|
| Key agreement | ECDH P-256 | ML-KEM-768 | ML-KEM-768 |
| Authentication | ECDSA P-256 | ML-DSA-65 | KEM implicit auth (no sig) |
| Certificates | ECC X.509 | PQ X.509 (DSA key) | PQ X.509 (KEM key) |
| Symmetric | AES-128-GCM | AES-256-GCM | AES-256-GCM |

KEMRSP's core argument is eliminating ML-DSA-65 entirely. Authentication is implicit via successful KEM decapsulation of the peer's long-term key (KEMTLS-style), which is where your bandwidth savings argument lands.

---

**Data sources**

- Classical: measured from actual `cryptography` library objects (real DER sizes)
- PQ-RSP and KEMRSP: NIST FIPS 203/204 specified constants
- Certificate overhead: X.509 ASN.1 shell ~200 bytes + key + signature

---

**Output tables for the paper**

1. Per-phase message size comparison (3 columns x 4 rows)
2. Key and cert size breakdown (isolated from BPP payload)
3. Total handshake bandwidth summary, excluding the ~25 KB BPP which is constant across all three

---

**What makes KEMRSP compelling in the results**

ML-DSA-65 signatures are 3,293 bytes each and appear 3--4 times across the handshake in PQ-RSP. KEMRSP replaces those with ML-KEM-768 ciphertexts (1,088 bytes) or eliminates the signature round entirely where implicit auth suffices. Net savings land around 6--8 KB per session, which is the key argument for constrained SGP.32/IoT RSP deployments.

---

Want to start with the Python simulation, the ProVerif models, or the Mermaid sequence diagrams?