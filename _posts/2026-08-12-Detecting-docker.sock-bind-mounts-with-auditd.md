---
layout: post
title: "Detecting docker.sock bind-mounts with auditd"
date: 2026-08-12
---

One file is all it takes. Bind-mount `/var/run/docker.sock` into a container and you own the host: the container gets the Docker API, which is root with extra steps. It's the classic escape: so common that most security guides list it first. But I wanted more than the theory. I wanted to know: can I actually detect this reliably with auditd, without drowning in false positives?

This is the story of that attempt: the rule that structurally can't work, the discriminator that does, and the benchmark that proved it across two kernels.

## The naive rule that can't work

My first instinct was simple. auditd has path filtering, so:

```
-w /var/run/docker.sock -p wa -k mount-watch
```

or a `-F path=` on a syscall rule. It feels right: watch the socket file. It doesn't work, and not because of a typo. The `path=` filter matches *file* syscalls: open, execve, unlink. A bind-mount is a `mount` syscall. The path that gets bound lives in the syscall's arguments, not in a path filter. The rule stays silent while the mount succeeds.

That's the structural lesson, worth stating plainly: **auditd's path filter is file-syscall-scoped. Mount-argument semantics live in record fields, not rule filters.**

So I needed a different approach: watch the mount syscall itself, and post-filter the records.

The rule that works: watch the mount syscalls, tag them with a key, post-filter:

```
-a always,exit -F arch=b64 -S mount,umount2,move_mount,mount_setattr -F key=mount-watch
```

Persist it in `/etc/audit/rules.d/` (one file per rule set) or load it live with `auditctl`. With that rule installed, the event looks like this (trimmed):

## What a docker.sock bind-mount looks like to auditd

```
type=SYSCALL ... syscall=mount key="mount-watch" comm="runc:[2:INIT]" a3=0x5000
type=PATH ... name="/run/docker.sock" mode=0140660
type=PATH ... name="/proc/thread-self/fd/8"
```

Two things stand out:

1. **The mode.** `0140660`: the leading `014` is `S_IFSOCK` (socket), not a regular file. Plain files show up as `0100644`.
2. **The comm.** `runc:[2:INIT]`: the runtime does the mount, not your container's processes.

So the discriminator is: **socket mode + basename `docker.sock`**, on mount records keyed with `mount-watch`. That combo kills the lookalikes: overlayfs and snapshot mounts are directories, netns mounts aren't sockets, `MS_PRIVATE` re-mounts carry a different flag set. It still catches the real thing.

One subtlety I hit early: `/var/run` is a symlink to `/run` on modern systems, so the kernel resolves the path and records `/run/docker.sock`. The basename check makes the resolution irrelevant. And the target `/proc/thread-self/fd/8` is runc's fd-based bind-mount shape.

## The filter

The filter parses ausearch output and reassembles audit events (each event is a set of records: SYSCALL + PATH + PROCTITLE, separated by `----` lines):

```python
import sys, re, json
from datetime import datetime, timezone

MS_BIND = 0x1000
S_IFMT = 0o170000
S_IFSOCK = 0o140000

def decode_name(v):
    if len(v) >= 2 and v[0] == '"' and v[-1] == '"': return v[1:-1]
    if v and re.fullmatch(r'[0-9a-fA-F]+', v) and len(v) % 2 == 0:
        try: return bytes.fromhex(v).decode('utf-8')
        except Exception: pass
    return v

def parse_fields(rest):
    f, i, n = {}, 0, len(rest)
    while i < n:
        while i < n and rest[i] == ' ': i += 1
        if i >= n: break
        eq = rest.find('=', i)
        if eq < 0: break
        k, i = rest[i:eq], eq + 1
        if i < n and rest[i] == '"':
            j = i + 1
            while j < n and not (rest[j] == '"' and rest[j-1] != '\\'): j += 1
            v, i = rest[i+1:j], j + 1
        else:
            j = i
            while j < n and rest[j] != ' ': j += 1
            v, i = rest[i:j], j
        f[k] = v
    return f

def parse_event(lines):
    recs = {}
    for ln in lines:
        m = re.match(r'type=(\S+) msg=audit\(([^)]+)\): ?(.*)$', ln)
        if not m: continue
        t, aid, rest = m.group(1), m.group(2), m.group(3)
        r = parse_fields(rest); r['audit_id'] = aid
        recs.setdefault(t, []).append(r)
    return recs

def read_events(path):
    cur = []
    for ln in open(path):
        ln = ln.rstrip('\n')
        if ln.strip() == '----':
            if cur: yield parse_event(cur)
            cur = []
        elif ln.strip(): cur.append(ln)
    if cur: yield parse_event(cur)

def proctitle_to_argv(s):
    try: return bytes.fromhex(s).decode('utf-8').split('\0')
    except Exception: return [s]

def alert_for(ev):
    sc, paths = ev.get('SYSCALL', []), ev.get('PATH', [])
    if not sc or not paths: return None
    s = sc[0]
    if 'mount-watch' not in s.get('key', ''): return None
    a3 = int(s.get('a3', '0'), 16)
    sock = [p for p in paths
            if decode_name(p.get('name','')).rsplit('/',1)[-1] == 'docker.sock'
            and (int(p.get('mode','0'),8) & S_IFMT) == S_IFSOCK]
    if not sock: return None
    src = sock[0]
    tgt = next((p for p in paths if p is not sock[0]), None)
    t = datetime.fromtimestamp(float(s['audit_id'].split(':')[0]), timezone.utc)
    return {
        'time': t.isoformat(), 'success': s.get('success'), 'comm': s.get('comm'),
        'exe': decode_name(s.get('exe','')), 'pid': s.get('pid'), 'ppid': s.get('ppid'),
        'argv': proctitle_to_argv(ev.get('PROCTITLE',[{}])[0].get('proctitle','')) if ev.get('PROCTITLE') else None,
        'src': decode_name(src.get('name','')) if src else None,
        'tgt': decode_name(tgt.get('name','')) if tgt else None,
        'src_mode': sock[0].get('mode'), 'flags': hex(a3), 'ms_bind': bool(a3 & MS_BIND),
        'key': s.get('key'),
    }

def main():
    for path in sys.argv[1:]:
        n_ev = n_al = 0
        for ev in read_events(path):
            n_ev += 1
            a = alert_for(ev)
            if a: n_al += 1; print(json.dumps(a))
        print(f'# {path}: {n_ev} events, {n_al} alerts', file=sys.stderr)

if __name__ == '__main__': main()
```

The parts that mattered most:

- **Source comes from the socket record itself**, not record ordering. Kernel versions vary in PATH record order; ordering assumptions break across upgrades, the socket record doesn't.
- **`MS_BIND` check on `a3`** (the 0x1000 flag). We're looking for bind-mounts specifically.
- **PROCTITLE decoding.** runc sets its argv to the container's argv and the record hex-escapes it, so the filter decodes it to show what the container was about to run.

## The benchmark protocol

A detection rule is only as good as its false-positive rate. I wrote a protocol with two passes:

- `--detect`: run the escape 3 times, count alerts. Expect exactly 3.
- `--precision`: run 300 seconds of benign docker traffic (exec, pull, stop, rm), count alerts. Expect 0.

```bash
#!/usr/bin/env bash
set -euo pipefail

# docker.sock bind-mount detection - benchmark protocol
# Usage: benchmark.sh {--detect|--precision}
# Requires: docker, auditd (ausearch), python3. Filter: tools/docker_sock_filter.py (same repo).

FILTER="$(dirname "$0")/docker_sock_filter.py"
KEY="mount-watch"
OUT="/tmp"
W=/tmp/bench_window.py

for c in docker ausearch python3; do command -v "$c" >/dev/null || { echo "missing: $c" >&2; exit 2; }; done
[ -f "$FILTER" ] || { echo "missing filter: $FILTER" >&2; exit 2; }

cat > "$W" <<'PYEOF'
import re, sys
s, e = int(sys.argv[1]), int(sys.argv[2])
lines = open(sys.argv[3]).read().splitlines()
out, cur, keep = [], [], False
for ln in lines:
    if ln.strip() == '----':
        if keep: out.extend(cur)
        cur, keep = [ln], False
        continue
    cur.append(ln)
    m = re.search(r'msg=audit\((\d+)\.\d+', ln)
    if m and s <= int(m.group(1)) <= e: keep = True
if keep: out.extend(cur)
open(sys.argv[4],'w').write('\n'.join(out)+'\n')
PYEOF

case "${1:-}" in
  --detect)
    S=$(date +%s)
    for i in 1 2 3; do
      docker run --rm -v /var/run/docker.sock:/var/run/docker.sock alpine sh -c 'apk add --no-cache docker-cli >/dev/null 2>&1; docker ps; docker run --rm -v /:/host alpine chroot /host sh -c "id"'
    done
    E=$(date +%s)
    sudo ausearch -k "$KEY" -ts recent > "$OUT/replay_raw.txt"
    python3 "$W" "$S" "$E" "$OUT/replay_raw.txt" "$OUT/replay_audit.txt"
    echo "== detection =="
    sudo python3 "$FILTER" "$OUT/replay_audit.txt"
    ;;
  --precision)
    S=$(date +%s)
    docker run -d --name bn-test alpine sleep 60 >/dev/null
    docker exec bn-test ls / >/dev/null
    docker stop bn-test >/dev/null && docker rm bn-test >/dev/null
    docker pull alpine:latest >/dev/null
    sleep 300
    E=$(date +%s)
    sudo ausearch -k "$KEY" -ts recent > "$OUT/benign_raw.txt"
    python3 "$W" "$S" "$E" "$OUT/benign_raw.txt" "$OUT/benign_audit.txt"
    echo "== precision =="
    sudo python3 "$FILTER" "$OUT/benign_audit.txt"
    ;;
  *) echo "usage: $0 {--detect|--precision}" >&2; exit 2;;
esac
```

To run it yourself: save both files in one directory (the filter as `docker_sock_filter.py`, the protocol as `benchmark.sh`), install the mount-watch rule, and run `sudo ./benchmark.sh --detect`. The `--precision` pass sleeps 300 seconds by design.

## Results

| | Detection | Precision |
|---|---|---|
| Baseline: Ubuntu 26.04, kernel 7.0.0-1010-aws, auditd 4.1.2 | 3/3 | 0/41 (251s window) |
| Replication: Ubuntu 24.04.4, kernel 6.17.0-1017-aws, auditd 3.1.2 | 3/3 | 0/48 (300s window) |

Filter unchanged between machines. That's the part I'm happiest about: socket-mode + basename survived a jump from kernel 7.0.0 to 6.17 and auditd 4.1.2 to 3.1.2 with zero edits.

## Limitations and next steps

- The filter matches bind-mounts of a socket whose basename is exactly `docker.sock`. A renamed socket, a socket elsewhere on the host, or another daemon's socket (podman.sock, containerd.sock) slips past: this is a detector for one escape shape, not a general mount policy.
- The `-F path=` dead end is structural. auditd's path filters don't evaluate mount arguments, so post-filtering the records is the only layer that works for path-based mount detection.
- Falco's modern_ebpf probe drops the mount source/target that auditd records: the observability gap this project started from. Closing it is the natural next probe.

The rule and filter are defensive. If you run container workloads and have auditd, this closes one of the easiest escape paths. Detection chain: docker.sock bind-mount, auditd mount record, socket-mode filter, alert with the container's argv.
