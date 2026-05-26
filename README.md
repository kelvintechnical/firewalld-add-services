# Lab: Adding Services to Zones — Named Opens with `--permanent`

- **Series:** linux-ops-mastery — RHCSA Firewall
- **Subjects covered:** `firewall-cmd --add-service`, `--remove-service`, `--permanent`, `firewall-cmd --reload`, runtime vs permanent drift checks, verifying with `--list-services`, zone-scoped additions
- **Career arcs covered:** RHCSA (classic “enable http/https” tasks), RHCE (`ansible.posix.firewalld`), SRE (change windows + documented rollback), DevOps (infra-as-code parity with runtime), AI/MLOps (exposing FastAPI/TensorBoard by service name)
- **Prerequisite:** Zone vocabulary (Lab **firewalld-zones**) and running `firewalld`
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 baseline services · 2–3 runtime add/remove · 4 permanent + reload · 5 edge: service not found · 6 capstone + cleanup

---

## Objective

Opening ports by memorizing TCP numbers is fine until you have eleven microservices. `firewalld` **services** are curated **name → port/protocol** mappings so you can say `http` instead of `80/tcp`. This lab teaches the RHCSA rhythm:

1. Try in **runtime** (immediate).
2. Mirror into **permanent** configuration.
3. **`--reload`** to apply permanent into runtime without rebooting.
4. **Verify** and **remove** when done.

You will add `http` (and optionally `https`) to a lab zone (default `public`) and prove persistence across reload — then strip the additions in **Cleanup** so the VM returns to stock teaching state.

> **Lab safety note:** Adding services **does** widen exposure. Task 6 **removes** services from the permanent config and reloads — always complete Cleanup on shared lab VMs.

---

## Concept: A “Service” Is a Shortcut Bundle

```
Zone: public
  services: [ssh, http, https]
        │        │      │
        │        │      └─ maps to 443/tcp (and possibly others)
        │        └─ maps to 80/tcp
        └─ maps to 22/tcp (plus helpers depending on distro)

firewalld expands names to nft/iptables rules behind the scenes.
```

> **Why this matters:** Exam tasks prefer **service names** because they are harder to mistype than five-digit port numbers, and they survive well-known port trivia drift in your head.

---

## 📜 Why firewalld Services Exist — The Story

Static firewall tutorials used to read like phone books: “allow 80/tcp, 443/tcp, 853/tcp, …” — correct, but brittle. Real applications speak in **protocol names** humans grep for in `ps` output (`httpd`, `nginx`) while operators think “web traffic.”

`firewalld` encodes that translation as **XML service definitions** under `/usr/lib/firewalld/services/`. Distributions ship a large catalog; vendors extend it for products. Administrators reference the catalog with **`--add-service`** instead of hand-carving matches.

Since **RHEL 7+**, `firewalld` has been the supported default firewall daemon on Red Hat systems, with continuous refinement of service lists for common workloads. The RHCSA candidate benefits because **service names** compress exam time: one flag instead of multiple `--add-port` lines — though custom ports remain necessary (Lab 59).

> **The point of the story:** Services are documentation that compiles into rules — treat them like code modules, not magic strings.

---

## 👪 The Service Addition Family — Who Lives There

### By persistence layer

| Layer | Flag | Survives reload? |
|---|---|---|
| Runtime | (none) | Until reload/reboot unless mirrored |
| Permanent | `--permanent` | Written to `/etc/firewalld/...` |
| Applied permanent | `--reload` | Pushes permanent → runtime |

### By command

| Action | Example |
|---|---|
| Add runtime | `firewall-cmd --add-service=http` |
| Add permanent | `firewall-cmd --permanent --add-service=http` |
| Remove | `--remove-service=http` |
| List | `--list-services` |

### By verification

| Question | Command |
|---|---|
| What is allowed now? | `firewall-cmd --list-services` |
| What is saved for boot? | `firewall-cmd --permanent --list-services` |

> **The point of the family tree:** Always ask **both** layers — runtime for “what works now,” permanent for “what survives reboot.”

---

## 🔬 The Anatomy of `firewall-cmd --add-service=http` — In One Diagram

```
$ firewall-cmd --add-service=http
success

$ firewall-cmd --list-services
cockpit dhcpv6-client http ssh
                          ^^^^ newly expanded name at runtime
```

Behind the scenes (conceptual):

```
--add-service=http
   └─► lookup service XML "http"
         └─► inject allow rules for its port/proto set into zone public
```

> **Reading rule:** `success` only means the daemon accepted the request — still run `--list-services` to prove visibility.

---

## 📚 Service Addition Reference Table

| Task | Command | Notes |
|---|---|---|
| Runtime add | `firewall-cmd --zone=public --add-service=http` | Zone explicit for clarity |
| Permanent add | `firewall-cmd --permanent --zone=public --add-service=http` | Writes config |
| Reload | `firewall-cmd --reload` | Rebuilds runtime from permanent |
| Runtime remove | `firewall-cmd --remove-service=http` | Immediate |
| Permanent remove | `firewall-cmd --permanent --remove-service=http` | Survives after reload |
| Query | `firewall-cmd --query-service=http` | Installed vs enabled differ — use `--list-services` for enabled |

> **Rule one of adds:** If you only did runtime, **`reload` wipes you** unless permanent mirrored.

---

## 🧪 Extended Verification Playbook (Optional Depth)

| Situation | Command | What “good” looks like |
|---|---|---|
| Prove service exists in catalog | `firewall-cmd --get-services \| tr ' ' '\n' \| grep ^http$` | Confirms identifier spelling |
| See XML definition | `sed -n '1,80p' /usr/lib/firewalld/services/http.xml` | Shows underlying ports/helpers |
| Compare enabled vs installed | `systemctl is-enabled httpd` vs `firewall-cmd --query-service=http` | Daemon enablement ≠ firewall opening |
| Runtime query | `firewall-cmd --list-services` | Always your first post-change check |
| Permanent query | `firewall-cmd --permanent --list-services` | Must match runtime after `reload` |
| Reload smoke | `firewall-cmd --reload && firewall-cmd --list-services` | Single-line habit for scripts |
| Conflict hunting | `firewall-cmd --list-ports` | Sometimes admins open `80/tcp` *and* `http` — redundant but valid |
| Evidence bundle | `firewall-cmd --list-all --zone=$ZONE \| tee /tmp/fw-evidence.txt` | Wider snapshot than services-only |

Optional depth only — skip on exam day if time is tight, but revisit when writing real change plans.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | “Permanently enable `http`” is almost a stock phrase — master `--permanent && --reload`. |
| **RHCE candidate** | Ansible `service:` parameter maps 1:1 — understand persistence to avoid false greens. |
| **SRE / Platform** | Change tickets should list **both** commands and a rollback removal pair. |
| **DevOps** | CI should diff `firewall-cmd --permanent --list-all` against golden output. |
| **AI / MLOps** | Dashboards (Grafana/TensorBoard) often map to `http`/`https` services — name them, don’t guess ports. |

---

## 🔧 The 6 Tasks

> Uses **`public`** zone unless your VM requires another — adjust `--zone=` consistently.

---

### Task 1 — Baseline: list runtime vs permanent services on `public`

**Purpose:** Prove whether drift already exists before you touch anything.

```bash
sudo -i
ZONE=public
echo "RUNTIME:"; firewall-cmd --list-services --zone=$ZONE
echo "PERMANENT:"; firewall-cmd --permanent --list-services --zone=$ZONE
```

**Human-Readable Breakdown:** On a fresh VM these often match. Any mismatch is pre-existing technical debt — note it.

**Reading it left to right:** Without `--permanent`, you read **runtime**. With `--permanent`, you read **saved** configuration.

**The story:** Smart engineers screenshot this pair before every change window.

**Expected output:**

```text
RUNTIME:
cockpit dhcpv6-client ssh
PERMANENT:
cockpit dhcpv6-client ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--list-services` | Enabled services for zone |
| `--permanent` | Select persistent config namespace |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `FirewallD is not running` | `systemctl start firewalld` |

---

### Task 2 — Core A: add `http` at runtime only and verify

**Purpose:** Feel immediate effect without persistence yet.

```bash
firewall-cmd --zone=$ZONE --add-service=http
firewall-cmd --zone=$ZONE --list-services
```

**Human-Readable Breakdown:** `http` should appear immediately among services.

**Reading it left to right:** `--add-service` mutates runtime; `--list-services` proves it.

**The story:** Runtime-only is perfect for “can we test now?” — dangerous if you stop here on exam day.

**Expected output:**

```text
success
cockpit dhcpv6-client http ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--add-service=http` | Allow well-known web port bundle |
| `--zone=` | Scope change |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `INVALID_SERVICE` | Typo — try `firewall-cmd --get-services \| grep http` |

---

### Task 3 — Core B: remove runtime `http` to practice rollback motion

**Purpose:** Pair add/remove muscle before permanent layer.

```bash
firewall-cmd --zone=$ZONE --remove-service=http
firewall-cmd --zone=$ZONE --list-services
```

**Human-Readable Breakdown:** Removal should mirror baseline Task 1 runtime list.

**Reading it left to right:** `--remove-service` is symmetric to add.

**The story:** Rollback rehearsal reduces panic during exams — you already know the undo incantation.

**Expected output:**

```text
success
cockpit dhcpv6-client ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--remove-service` | Deletes runtime allowance |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `NOT_ENABLED` style message | Service wasn’t present — verify list |

---

### Task 4 — Permanent + reload: add `http` and `https`, then apply

**Purpose:** RHCSA-grade persistence pattern.

```bash
firewall-cmd --permanent --zone=$ZONE --add-service=http
firewall-cmd --permanent --zone=$ZONE --add-service=https
firewall-cmd --reload
echo "RUNTIME:"; firewall-cmd --list-services --zone=$ZONE
echo "PERMANENT:"; firewall-cmd --permanent --list-services --zone=$ZONE
```

**Human-Readable Breakdown:** Permanent lines stack; `reload` reconciles runtime to permanent without reboot.

**Reading it left to right:** Order is canonical: **write permanent** → **reload** → **verify both namespaces match**.

**The story:** Forgetting `reload` is how you pass the file edit but fail the live test — always verify runtime after.

**Expected output:**

```text
success
success
success
RUNTIME:
cockpit dhcpv6-client http https ssh
PERMANENT:
cockpit dhcpv6-client http https ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--permanent` | Target on-disk configuration |
| `--reload` | Rebuild active rules |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Runtime lacks `http` after reload | Permanent add failed — re-run with `echo $?` tracing |
| `reload` drops unrelated runtime tweaks | Expected — only permanent is authoritative after reload |

---

### Task 5 — Edge case: attempt bogus service + locate real service names

**Purpose:** Train error literacy and discovery with `--get-services`.

```bash
firewall-cmd --permanent --zone=$ZONE --add-service=fake-service-xyz 2>&1 | head
firewall-cmd --get-services | tr ' ' '\n' | grep -E '^(http|https|ssh)$'
```

**Human-Readable Breakdown:** First command should fail loudly; second proves discovery path.

**Reading it left to right:** stderr captured into stdout with `2>&1` for a single pipeline view.

**The story:** Autocomplete lies; `get-services` does not.

**Expected output:**

```text
Error: INVALID_SERVICE: fake-service-xyz
http
https
ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `--get-services` | Prints all known service identifiers |
| `2>&1` | Merge stderr for filtering |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Error text differs slightly | Normal across minor versions — intent matters |

---

### Task 6 — Capstone: document state, then strip `http`/`https` permanently

**Purpose:** Prove you can **return VM to teaching baseline** — exam proctors appreciate reversibility.

```bash
{
  date
  echo "RUNTIME SERVICES"; firewall-cmd --list-services --zone=$ZONE
  echo "PERMANENT SERVICES"; firewall-cmd --permanent --list-services --zone=$ZONE
} | tee /tmp/services-lab.txt
```

**Human-Readable Breakdown:** Capture final enabled state before removal for learning journal.

**The story:** Capstone is the story you tell: “I can add, persist, verify, and remove without orphan rules.”

**Expected output:**

```text
RUNTIME SERVICES
cockpit dhcpv6-client http https ssh
PERMANENT SERVICES
cockpit dhcpv6-client http https ssh
```

**Switches**

| Token | Meaning |
|---|---|
| `tee` | Saves evidence under `/tmp` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `http` missing | Skip removal or adjust Cleanup block |

**Cleanup**

```bash
firewall-cmd --permanent --zone=$ZONE --remove-service=http
firewall-cmd --permanent --zone=$ZONE --remove-service=https
firewall-cmd --reload
firewall-cmd --list-services --zone=$ZONE
firewall-cmd --permanent --list-services --zone=$ZONE
rm -f /tmp/services-lab.txt
```

---

## 🔍 Service Addition Decision Guide

```
Need to open a standard daemon port?
  │
  ├── "Does firewalld know a service name?"
  │       └── YES → --add-service (runtime) → mirror --permanent → --reload
  │
  ├── "Non-standard port?"
  │       └── Lab 59 firewalld-custom-ports
  │
  ├── "Temporary test only?"
  │       └── Runtime add without --permanent (know it vanishes on reload)
  │
  └── "Rollback?"
          └── --remove-service symmetrically in both layers
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Compare runtime vs permanent service lists on `public`
- [ ] 02 Add `http` runtime-only and verify with `--list-services`
- [ ] 03 Remove `http` runtime-only and verify return to baseline services
- [ ] 04 Add `http` + `https` with `--permanent`, run `--reload`, verify runtime/permanent parity
- [ ] 05 Provoke `INVALID_SERVICE`, then discover valid names with `--get-services`
- [ ] 06 Document with `tee`, then Cleanup permanent removals + reload + delete `/tmp` file

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot `--reload` | Reboot would work; live test fails expectation | Always reload after permanent edits |
| Only permanent, never runtime check | False confidence | Compare lists post-reload |
| Typo service | INVALID_SERVICE | `firewall-cmd --get-services` |
| Removed `ssh` accidentally | Lockout | Console; add `ssh` back |
| Mixed zones | Changes appear “missing” | Always pass explicit `--zone=` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the three-line incantation: `permanent --add-service` → `reload` → `list-services` twice.

**RHCE candidate**
- Explain why `immediate:` vs `permanent:` Ansible args mirror these CLI flags.

**SRE / Platform interview**
- Discuss blast radius: adding `https` opens TLS — still not application auth.

**DevOps**
- Store generated zone XML in Git when you must diverge from stock services.

**AI / MLOps**
- Expose metrics on `:9090`? That likely needs **custom port** — link Labs 58→59.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [firewalld-custom-ports](https://github.com/kelvintechnical/firewalld-custom-ports) | When no service XML exists |
| [firewalld-zones](https://github.com/kelvintechnical/firewalld-zones) | Choosing correct zone before adding services |
| [default-firewall-zone](https://github.com/kelvintechnical/default-firewall-zone) | Default vs explicit zone context |
| [inspecting-iptables](https://github.com/kelvintechnical/inspecting-iptables) | Seeing resulting chain counters |

---

## 🎓 After the Lab — 60-Second Oral Exam

Answer out loud without scrolling:

- What is the difference between **runtime** and **`--permanent`** service lists?
- Why is `firewall-cmd --reload` required after permanent edits?
- What error appears when you typo a service name?
- Name the symmetric command pair to undo a service add.
- After reload, which two `list-services` commands must agree?
- What is the difference between `--query-service` and `--list-services`?

If any answer wobbles, redo Tasks 2–4 slowly — persistence is the graded muscle.

> **Proctor hint:** Whisper **permanent → reload → verify** as one three-beat mantra before you touch the keyboard.

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
