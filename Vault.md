# Ansible Vault

```/.vault_pass sama /host_vars ```

## Step 1 — Buat vault password file

```
# Di ansible-control
echo "your_vault_master_password" > ~/.vault_pass
chmod 600 ~/.vault_pass
```

```Ini adalah master password untuk decrypt vault```

## Step 2 — Encrypt sudo password

```
ansible-vault encrypt_string 'your_sudo_password' \
  --name 'ansible_become_pass' \
  --vault-password-file ~/.vault_pass
```

Output akan seperti ini:

```
ansible_become_pass: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  61383932343634613538616637323837...
```

Copy output tersebut.

## Step 3 — Simpan di ```group_vars```

```~/ansible/group_vars```

Karena setiap node punya user berbeda, buat per-host vars:

```
# Struktur
~/ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.yml
└── group_vars/
    └── all.yml        # kalau sudo password semua node sama
# ATAU
└── host_vars/
    ├── nlab-1.yml     # kalau beda per node
    └── nlab-3.yml
```

## Step 4 — Update ansible.cfg

```
[defaults]
inventory         = inventory/hosts.yml
private_key_file  = ~/.ssh/ansible_lab
host_key_checking = False
stdout_callback   = yaml
callbacks_enabled = timer, profile_tasks
vault_password_file = ~/.vault_pass

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root
```

### Hanya orang yang punya ~/.vault_pass yang bisa decrypt.