# Ansible Demonstration — Run Order

**Theme:** Brush (ansible control node) and Canvas (managed host).
**Goal:** Show the same Linux admin tasks done first **by hand**, then **automated with ansible** — so the audience understands what ansible is actually doing under the hood.

| Host | IP | Role |
|---|---|---|
| brush | 192.168.50.5 | Ansible control node |
| canvas | 192.168.50.26 | Managed host (target of all changes) |

---

## Pedagogical order (why this order matters)

1. **Manual first** — audience builds a mental model of *what* needs to happen (groups, users, files, modes).
2. **Automation second** — audience sees ansible automate the same work, then go further with a full service install (packages, DB, vhost, reload) in one command. Without the manual phase first, ansible looks like magic instead of automation.

The manual phase covers users, groups, and permissions only — the service install (Piwigo) is ansible-only. The manual phase uses a parallel namespace (`painters` group, `/srv/atelier`) so it doesn't collide with the ansible-managed state (`artists` group, `/srv/studio`).

> **Cleanup flags for `reset.yml`:**
> - default — wipes ansible side only (`artists`, vangogh/etc., `/srv/studio`)
> - `-e reset_piwigo=true` — also wipes the ansible Piwigo (DB + files + vhost)
> - `-e clear_all=true` — wipes **everything**: ansible side + Piwigo + manual side (`painters`, rembrandt/etc., `/srv/atelier`)

---

## Pre-demo prep (do BEFORE audience arrives)

```bash
# 1. Verify both VMs are up and you can SSH
ssh root@192.168.50.5  hostname     # → brush
ssh root@192.168.50.26 hostname     # → canvas

# 2. Wipe canvas (ansible side + manual side + Piwigo) — single command
ssh root@192.168.50.5 'cd /root/ansible && ansible-playbook reset.yml -e clear_all=true'

# 3. Open two terminal tabs/windows on your laptop:
#    - Tab A: ssh root@192.168.50.26   (canvas — for manual phase)
#    - Tab B: ssh root@192.168.50.5    (brush  — for ansible phase)
#    Use a large readable font.

# 4. Open a browser tab to:  http://192.168.50.26/install.php   (Piwigo wizard, will hit at the end)
```

---

# PHASE 1 — Manual administration (on canvas)

> **Tab A — `ssh root@192.168.50.26`**
> **Narration:** "First I'll show what a sysadmin does by hand. Watch every command — every one of them is a line of code we'll later see ansible take care of for us."

## 1.1  Group and users

```bash
# Show that the group does not exist yet
getent group painters         # → no output

# Create the group
groupadd painters
getent group painters         # → painters:x:1001:

# Create six artist users, all in the painters group, each with a home dir
for u in rembrandt klimt dali kahlo banksy okeeffe; do
    useradd -m -s /bin/bash -g painters "$u"
done

# Verify
getent group painters
id rembrandt
ls -ld /home/rembrandt
```

**Talking point:** "Six commands per user the long way, or one `for` loop. Either way, six users. Now imagine doing this on 200 servers."

## 1.2  Shared studio directory and per-user permissions

```bash
mkdir -p /srv/atelier
chown root:painters /srv/atelier
chmod 2775 /srv/atelier            # 2 = setgid bit → new files inherit 'painters' group

# Per-user subdirectory
for u in rembrandt klimt dali kahlo banksy okeeffe; do
    install -d -o "$u" -g painters -m 0750 /srv/atelier/"$u"

    # Private file (mode 600 — only owner can read)
    echo "Private sketches by $u." > /srv/atelier/"$u"/sketches.txt
    chown "$u":painters /srv/atelier/"$u"/sketches.txt
    chmod 600 /srv/atelier/"$u"/sketches.txt

    # Public file (mode 644 — world-readable)
    echo "Public portfolio of $u." > /srv/atelier/"$u"/portfolio.txt
    chown "$u":painters /srv/atelier/"$u"/portfolio.txt
    chmod 644 /srv/atelier/"$u"/portfolio.txt
done

# Shared file the whole group can edit (mode 664)
cat > /srv/atelier/shared-masterpiece.txt <<'EOF'
Collaborative canvas. Members of 'painters' can write here.
----
EOF
chown root:painters /srv/atelier/shared-masterpiece.txt
chmod 664 /srv/atelier/shared-masterpiece.txt

ls -lR /srv/atelier
```

**Talking point:** Walk through `ls -l` output column by column — owner, group, mode. Point out the `s` in `drwxr-sr-x` (setgid).

## 1.3  Live permission proof

```bash
# Owner can read their own private file
sudo -u rembrandt cat /srv/atelier/rembrandt/sketches.txt
#   → "Private sketches by rembrandt."

# Another group member CANNOT read it (mode 600)
sudo -u klimt cat /srv/atelier/rembrandt/sketches.txt
#   → cat: ...: Permission denied

# But the shared file is group-writable (mode 664)
sudo -u klimt tee -a /srv/atelier/shared-masterpiece.txt <<<"klimt was here"
cat /srv/atelier/shared-masterpiece.txt
```

**Talking point:** "This is the difference between 600 and 664 in one screen. Linux permissions aren't theoretical — `Permission denied` is the kernel enforcing them in real time."

> **Bridge to Phase 2:** "Doing the user/perm setup by hand is tedious but doable. Now imagine doing the same thing on 200 servers — and on top of that, installing a full service: packages, database, vhost, the works. That's where ansible earns its keep."

---

# PHASE 2 — Automation with ansible (on brush)

> **Tab B — `ssh root@192.168.50.5`**
> **Narration:** "Now we'll do that same set of steps from a single command. The work happens *on canvas*, but I'm sitting on brush — I never log into canvas during this phase."

## 2.1  Show the workspace

```bash
cd /root/ansible
ls
#   ansible.cfg  artists.yml  inventory.ini  piwigo.yml  reset.yml
```

## 2.2  Walk through the inventory and config

```bash
cat inventory.ini       # the list of hosts ansible can talk to
cat ansible.cfg         # default settings
```

**Talking point:** "The inventory is just a text file. There's one host in it: `canvas`. That's the only place I name the target."

## 2.3  Walk through the playbook

```bash
cat artists.yml
```

**Talking point:** Map each ansible task back to a manual command from Phase 1. "This `ansible.builtin.group` task — that's the `groupadd` from earlier. This `ansible.builtin.user` loop — that's the six `useradd` commands. This `file` task — that's `mkdir + chown + chmod` in one."

## 2.4  Prove canvas is empty (the artists side)

```bash
ssh root@192.168.50.26 'getent group artists; ls /srv/studio 2>&1'
#   → no artists group, "No such file or directory"
```

## 2.5  Dry run — show what *would* happen

```bash
ansible-playbook artists.yml --check --diff
```

**Talking point:** "`--check` is ansible's plan mode. Nothing actually happens — it just tells you what it would do. This is how sysadmins sanity-check changes before letting them touch production."

## 2.6  Real run, step-by-step

```bash
ansible-playbook artists.yml --step
```

Press `y` after each task; pause to narrate what just happened.

**Talking point at each task:**
- *group* — "That created the `artists` group. Same as `groupadd`."
- *user loop* — "Six users in one task. Same as the `for` loop earlier."
- *file (studio root)* — "Directory + owner + group + mode in one declaration."
- *copy (sketches)* — "Drops a file with the right content, owner, and 600 mode in one go."
- *Print layout* — "Free verification — ansible can show you the result."

## 2.7  Verify on canvas

```bash
ssh root@192.168.50.26 'ls -lR /srv/studio | head -30'
```

## 2.8  Now the service playbook

```bash
ansible-playbook piwigo.yml
```

**Talking point:** "All the moving parts of a service install — install 11 packages, create the database, create a database user, download the app, extract it, set ownership, write an apache vhost, enable a module, enable the site, reload the service — collapse into a single command. By hand that's ten distinct steps and a lot of room to typo. The playbook is also reusable on any number of servers."

When it's done, open in browser:
```
http://192.168.50.26/install.php
```
Walk the audience through the Piwigo install wizard:
- Host: `localhost`
- User: `piwigo`
- Password: `BrushAndCanvas!2026`
- Database: `piwigo`

---

## 2.9  Idempotency demo (the killer feature)

Re-run the artists playbook:
```bash
ansible-playbook artists.yml
```

**Talking point:** "Notice every task says `ok` instead of `changed`. Ansible is *idempotent* — it only does work if work is needed. Run it a hundred times, the result is identical. Try doing that with a bash script."

---

# Closing comparison

Top rows are what the audience saw run live (Phase 1 manual ↔ Phase 2 ansible). Bottom rows (italics) are what `piwigo.yml` would equate to *if* you'd done the service install by hand — included to show the breadth of what one playbook collapses.

| Task | Manual | Ansible |
|---|---|---|
| Create group | `groupadd painters` | `group: name=artists` |
| Create 6 users | `for u in …; do useradd -m -g painters "$u"; done` | `user:` task + `loop:` |
| Directory + mode | `mkdir; chown; chmod` (3 cmds) | `file: state=directory owner= group= mode=` (1 task) |
| File with content + mode | `echo > … ; chown … ; chmod …` (3 cmds) | `copy: dest= content= owner= group= mode=` (1 task) |
| *Install 11 packages* | *`apt install -y pkg1 pkg2 …`* | *`apt: name=[…]`* |
| *Create DB + user* | *`mysql <<SQL … SQL`* | *`mysql_db` + `mysql_user`* |
| *Reload service* | *`systemctl reload apache2`* | *handler: `service: state=reloaded`* |
| **Re-running** | `Error: group already exists` | `ok` (idempotent) |
| **Across N servers** | SSH into each one and repeat | Add hostnames to inventory, done |

---

# Post-demo cleanup (optional)

If you want to leave canvas in a clean state for the next demo:

```bash
ssh root@192.168.50.5 'cd /root/ansible && ansible-playbook reset.yml -e clear_all=true'
```

---

# Cheat sheet — flags worth knowing during Q&A

| Flag | What it does |
|---|---|
| `--check` | Dry run — show changes without making them |
| `--diff` | Show file content diffs alongside `--check` |
| `--step` | Pause for confirmation before every task |
| `-v` / `-vv` / `-vvv` | Verbose output (more `v`s = more detail) |
| `--start-at-task "Name"` | Skip ahead to a specific task |
| `--limit canvas` | Restrict to one host from the inventory |
| `--tags users --skip-tags piwigo` | Cherry-pick tasks (requires `tags:` on tasks) |
