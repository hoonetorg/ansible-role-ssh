# ansible-role-ssh

SSH host keys and known hosts on openSUSE Leap, from the inventory:

1. **Host keys**: the private keys from the vault go to `/etc/ssh/ssh_host_<type>_key` (root, 0600), the public
   keys to `.pub` (0644) → after a reinstall the host has its old keys again, nobody gets a "REMOTE HOST
   IDENTIFICATION HAS CHANGED". sshd is reloaded on changes (if it runs). Types not listed are not touched; without
   private keys nothing is touched.
2. **`/etc/ssh/ssh_known_hosts`**: exactly the entries of one list (the same file on every host): your hosts and
   other static machines. The inventory is not scanned: a host is in the file only if it has an entry. Generated
   as a whole (desired state); the old file is kept as backup on changes.
3. **Temporary VMs** (virtbuilder, libvirt networks): `/etc/ssh/ssh_config.d/50-vm-known-hosts.conf` sends the
   listed patterns to `~/.ssh/known_hosts_vms` of each user, new keys accepted without asking. All other hosts keep
   using `~/.ssh/known_hosts` as before.

## How ssh checks host keys

ssh loads `~/.ssh/known_hosts` and `/etc/ssh/ssh_known_hosts` together. If **any** of them has the key the server
shows, the host is accepted - also when the other file still holds an old key. ssh refuses only if no file has the
key but one has another key for the name ("HOST IDENTIFICATION HAS CHANGED"); it then offers to remove the entry
only from the user file.

That was the problem of a hand-made global file: when a key changed, the stale entry in the root-owned file could
not be fixed by the user. Here the global file is generated from the inventory: a key rotation is an inventory
change, the host gets its new keys and every other host its new global file in the next run. An old entry in a
user file then doesn't matter (the global file has the right key).

Only machines whose key changes you notice belong in the list (your hosts, servers you manage). Everything else
stays in `~/.ssh/known_hosts`, where you can accept/remove keys yourself.

## Variables

```yaml
ssh:
  host_keys:
    private: "{{ vault_ssh_host_private_keys }}"   # {ed25519: "-----BEGIN ...", ecdsa: ..., rsa: ...}; {} = untouched
  known_hosts:                                     # [] = /etc/ssh/ssh_known_hosts not touched
    - names: [de1, de1.example.org, 192.168.10.20] # inventory host: its inventory name is in names
      keys:
        ed25519: "ssh-ed25519 AAAA... root@de1"
        ecdsa: "ecdsa-sha2-nistp256 AAAA... root@de1"
        rsa: "ssh-rsa AAAA... root@de1"
    - names: [server1, 192.168.10.5]               # static machine outside the inventory
      keys:
        ed25519: "ssh-ed25519 AAAA..."
  vm_hosts: ["192.168.122.*", "*.vm.example.org", "osl"]   # [] = no file
  packages: [openssh-server, openssh-clients]
```

The public keys of the host itself come from the `known_hosts` entry whose `names` contain the inventory name; a
private key needs its public key there. Entry names: no spaces or commas; the key types ed25519, ecdsa, rsa; the
comment of a key (`root@...`) is written to `.pub`, not to the global file.

`vm_hosts` are ssh `Host` patterns. ssh compares them with what you type, not with the resolved address: list the
IP pattern and the names (`ssh osl` doesn't match `192.168.123.*`). No CIDR, only `*`/`?` wildcards.

## Getting the current keys into the inventory

Before the first run (or before a reinstall) take the keys of the running machine, so nothing changes:

```bash
# public keys → an entry in the known_hosts list
cat /etc/ssh/ssh_host_{ed25519,ecdsa,rsa}_key.pub
# private keys → YAML to paste into `ansible-vault edit host_vars/<host>/vault.yml`
sudo sh -c 'echo "vault_ssh_host_private_keys:"; for t in ed25519 ecdsa rsa; do echo "  $t: |"; sed "s/^/    /" /etc/ssh/ssh_host_${t}_key; done'
```

The first run then reports no change for the host keys.

**Rotation:** new keys with `ssh-keygen -t <type> -N '' -C root@<host> -f /tmp/k` (somewhere safe, delete it after),
private key into the vault, public key into the list, run on all hosts.

## Safety

- The private keys are never shown: the merged configuration (`ssh_final`) is not logged, the key files are
  written without output and diff, loops run over the type names only.
- A private key that doesn't belong to its public key in the list is not written (`validate` with
  `ssh-keygen -y`); also an unchanged key file is checked against the list. The run stops with the type name, the
  `.pub` file is not written.
- Replaced key files are kept as backup `/etc/ssh/ssh_host_<type>_key.<timestamp>~` (root, 0600; sshd doesn't load
  them). These are old private keys: check, then delete them.

## Temporary VMs

A rebuilt VM with the same IP/name has a new key; `accept-new` only accepts unknown hosts, so ssh refuses. Remove
the old entry:

```bash
ssh-keygen -R <ip|name> -f ~/.ssh/known_hosts_vms   # one VM (keeps known_hosts_vms.old)
: > ~/.ssh/known_hosts_vms                         # all VMs: empties the file, it only holds VM keys
```

## Check

```bash
cat /etc/ssh/ssh_known_hosts
ssh -G <host> | grep -i -E 'userknownhostsfile|stricthostkeychecking'   # VM pattern or normal host
ls -lZ /etc/ssh/ssh_host_*                     # 0600/0644, SELinux sshd_key_t
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # fingerprint
```

## License

Apache-2.0

Created with the help of AI
