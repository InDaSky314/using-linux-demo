# Using Linux — Manually and Automated

Lab materials for a school Career Practicum demonstration: the same Linux administration
work performed first by hand, then with Ansible. Two Ubuntu VMs, one shared studio
directory full of permission bits, and a full Piwigo install collapsed from ten manual
steps to one playbook.

```
┌──────────────────────┐         ┌──────────────────────┐
│ brush                │  SSH    │ canvas               │
│ 192.168.50.5         ├────────▶│ 192.168.50.26        │
│ Ansible control node │         │ Managed host         │
│ /root/ansible/       │         │ sshd + python3 only  │
└──────────────────────┘         └──────────────────────┘
```

## What's in this repo

```
ansible/        playbooks that run from brush against canvas
  ansible.cfg
  inventory.ini             one host: canvas
  artists.yml               creates artists group + 6 users + /srv/studio
  piwigo.yml                installs Apache + MariaDB + PHP + Piwigo
  reset.yml                 teardown — supports clear_all=true for full nuke
docs/
  demo-run-order.md         the full demo run-of-show, narration included
presentation/
  Using_Linux_Manually_and_Automated.pptx   19-slide audience deck
  build_deck.py             python-pptx generator — re-run to rebuild .pptx
```

## Lab setup

Two Ubuntu 24.04 VMs on the same /24:

| Host   | IP             | Role               | Needs installed             |
|--------|----------------|--------------------|-----------------------------|
| brush  | 192.168.50.5   | Ansible controller | `ansible`, `python3`, sshd  |
| canvas | 192.168.50.26  | Managed host       | sshd, `python3` (that's it) |

Both have root SSH key-auth (the demo uses `ansible_user=root` in
`ansible/inventory.ini`). Adjust IPs in `inventory.ini` and the demo doc if your
hosts live elsewhere.

## Running the demo

The full narration + command sequence is in [docs/demo-run-order.md](docs/demo-run-order.md).
Short version:

```bash
# 0. Reset canvas to a clean slate (wipes ansible side + manual side + Piwigo)
ssh root@192.168.50.5 'cd /root/ansible && ansible-playbook reset.yml -e clear_all=true'

# 1. Phase 1 — by hand on canvas (open ssh root@192.168.50.26 in one tab)
#    Run §1.1–1.3 from docs/demo-run-order.md: groupadd, useradd loop,
#    chmod, install -d, sudo -u rembrandt vs sudo -u klimt for the live
#    permission-denied moment.

# 2. Phase 2 — ansible on brush (open ssh root@192.168.50.5 in the other tab)
ansible-playbook artists.yml --check --diff   # dry run
ansible-playbook artists.yml --step           # real, paced
ansible-playbook artists.yml                  # idempotency demo: ok=8 changed=0
ansible-playbook piwigo.yml                   # full service install

# 3. Open http://192.168.50.26/install.php and walk the Piwigo wizard
#    (DB user: piwigo, password: BrushAndCanvas!2026, DB: piwigo)
```

## reset.yml flags

| Invocation                                     | Wipes                                          |
|------------------------------------------------|------------------------------------------------|
| `ansible-playbook reset.yml`                   | ansible side only (artists, /srv/studio)       |
| `ansible-playbook reset.yml -e reset_piwigo=true` | + Piwigo (DB, files, vhost)                |
| `ansible-playbook reset.yml -e clear_all=true` | **everything**: + manual side (painters, /srv/atelier) |

## Rebuilding the slide deck

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install python-pptx
python3 presentation/build_deck.py
```

The deck is generated programmatically (not hand-edited in PowerPoint) so the source of
truth is `presentation/build_deck.py`. The .pptx in the repo is the most recent build
checked in for convenience.

## Two parallel namespaces

The demo deliberately uses different names for the manual and ansible halves so they
don't collide:

| Concept       | Manual (Phase 1) | Ansible (Phase 2) |
|---------------|------------------|-------------------|
| Group         | `painters`       | `artists`         |
| Users         | rembrandt, klimt, dali, kahlo, banksy, okeeffe | vangogh, picasso, davinci, monet, frida, warhol |
| Studio root   | `/srv/atelier`   | `/srv/studio`     |

## Notes

- Piwigo DB password (`BrushAndCanvas!2026`) is a lab credential. Don't reuse it.
- The deck's visual style (dark background, green accent, Calibri + Consolas) was
  derived from a separate `Linux_From_The_Ground_Up.pptx` umbrella deck so the demo
  slot matches the rest of the practicum series.
- `ansible-demo-notes.md` (now `docs/demo-run-order.md`) is the canonical run-of-show.
  Update it first when the demo changes.
