# Parse /etc/passwd and /etc/group — List Users by Group

**Difficulty:** Medium
**Topic:** [[sre/topics/linux-cli]]
**Pattern:** File Parsing / Join / Hash Map
**Companies:** [[sre/companies/apple]], [[sre/companies/meta]], [[sre/companies/google]]

## Problem

Write a script that reads `/etc/passwd` and `/etc/group` and prints, for each group, a sorted list of all member usernames — including users whose **primary GID** matches the group (from `/etc/passwd`) as well as users listed as **supplementary members** (from `/etc/group`).

Expected output format:
```
adm: root, syslog
docker: alice, bob
staff: alice, bob, charlie
sudo: alice
wheel: root
```

## Why This Is Tricky

Users belong to a group in **two different ways**:

1. **Primary group:** The GID in the 4th field of `/etc/passwd`. This is *not* listed in `/etc/group`'s members field — you must infer it.
2. **Supplementary groups:** Listed explicitly in `/etc/group`'s members field.

A naive script that only reads `/etc/group` members misses all primary-group memberships. Most interviews catch this in a follow-up.

### File Formats

```
# /etc/passwd: username:password:UID:GID:GECOS:home:shell
root:x:0:0:root:/root:/bin/bash
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
bob:x:1002:1002:Bob Jones:/home/bob:/bin/sh

# /etc/group: groupname:password:GID:member1,member2,...
root:x:0:
staff:x:50:alice,bob,charlie
docker:x:999:alice,bob
alice:x:1001:          # alice's primary group — no members listed here
bob:x:1002:            # bob's primary group
```

## Approach

1. Parse `/etc/passwd` → build `username → primary_gid` map.
2. Parse `/etc/group` → build `gid → group_name` map and `gid → set(supplementary_members)`.
3. For every user in `/etc/passwd`, add them to the `group_members[primary_gid]` set.
4. Merge supplementary members into the same sets.
5. Print each group with its sorted member list.

## Solution (Python)

```python
from collections import defaultdict
import sys

def list_users_by_group(passwd_path="/etc/passwd", group_path="/etc/group"):
    primary_gid = {}          # username -> primary GID string
    gid_to_name = {}          # GID string -> group name
    members = defaultdict(set) # GID string -> set of usernames

    # Pass 1: /etc/passwd — collect username and primary GID
    with open(passwd_path) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            parts = line.split(':')
            if len(parts) < 4:
                continue
            username, gid = parts[0], parts[3]
            primary_gid[username] = gid

    # Pass 2: /etc/group — collect group names and supplementary members
    with open(group_path) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            parts = line.split(':')
            if len(parts) < 3:
                continue
            group_name, gid = parts[0], parts[2]
            gid_to_name[gid] = group_name
            # Supplementary members are in field 4 (may be absent or empty)
            if len(parts) >= 4 and parts[3].strip():
                for user in parts[3].split(','):
                    user = user.strip()
                    if user:
                        members[gid].add(user)

    # Merge primary group memberships (the key gotcha)
    for username, gid in primary_gid.items():
        members[gid].add(username)

    # Print sorted by group name
    for gid, group_name in sorted(gid_to_name.items(), key=lambda x: x[1]):
        group_members = sorted(members.get(gid, set()))
        if group_members:
            print(f"{group_name}: {', '.join(group_members)}")

if __name__ == "__main__":
    passwd = sys.argv[1] if len(sys.argv) > 1 else "/etc/passwd"
    group  = sys.argv[2] if len(sys.argv) > 2 else "/etc/group"
    list_users_by_group(passwd, group)
```

## Solution (Bash one-liner variant — supplementary only)

```bash
# WARNING: This only shows supplementary members, misses primary-group membership
awk -F: 'NF>=4 && $4!="" {print $1": "$4}' /etc/group | sort
```

Use the Python version for correctness in an interview. Mention the bash limitation.

## Complexity

- **Time:** O(U + G) where U = users in passwd, G = groups in group file.
- **Space:** O(U + G) for the maps.

## Key Insight

The `/etc/passwd` GID and `/etc/group` members field are **two separate mechanisms** for group membership. You must combine both to get the complete picture. This mirrors a real operational concern: `id <username>` shows groups from both sources, while `cat /etc/group | grep <user>` only shows supplementary.

## Variants / Follow-ups

- "Filter out system accounts (UID < 1000)" → add `uid = int(parts[2]); if uid < 1000: continue`.
- "Output as JSON" → collect into a dict, print via `json.dumps`.
- "Handle corrupt lines gracefully" → add `try/except ValueError` around `int()` calls.
- "What if the files are 1GB?" → They won't be (no machine has that many users), but if asked: stream line-by-line (already done) and ask whether the maps fit in RAM.
- "What does `getent group` do that this doesn't?" → `getent` queries NSS (LDAP, NIS, SSSD), not just flat files. This script only covers local accounts.

## Sources

- [[sre/companies/apple]]
- [[sre/companies/meta]]
