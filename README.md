# Ansible Core Concepts — Hands-on Lab

## === Structure Folder ===

```text
C:.
|   .codex
|   .vault_pass
|   ansible.cfg
|   Vault.md
|
+---inventory
|   |   hosts.yml
|   |
|   \---host_vars
|           nlab-1.yaml
|           nlab-1.yaml.bak
|           nlab-3.yaml
|           nlab-4.yaml
|
\---playbooks
        debug_become.yaml
        htop.yaml
        variables.md
        variables.yaml
```

## === SSH AND IP ADDRESS ===

Pastikan IP Address Static, bisa setting pake 'netplan'.

Disetiap VM Worker harus dipasang SSH KEY dari Master Controller.

**Generate SSH di Master -> Pasang ke Worker**

- **Generate Key (Master)**
  ```bash
  ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/ansible_lab -N ""
  ```

- **Pasang Key (Worker)**
  ```bash
  ssh-copy-id -i ~/.ssh/ansible_lab.pub nlab-1@192.168.1.101
  ```

- **Testing Key (Master)**
  ```bash
  ssh -i ~/.ssh/ansible_lab nlab-1@192.168.1.101
  ```

---

## === Core Concepts ===

### 1. Ansible Inventory

Daftar hosts yang dikelola Ansible. Bisa format INI atau YAML.

**YAML Format**
```yaml
all:
  children:
    managed_nodes:
      hosts:
        nlab-1:
          ansible_host: 192.168.1.101
          ansible_user: nlab-1
        nlab-3:
          ansible_host: 192.168.1.103
          ansible_user: nlab-3
        nlab-4:
          ansible_host: 192.168.1.104
          ansible_user: nlab-4
```

**INI Format**
```ini
[all.children.managed_nodes.hosts.nlab-1]
ansible_host=192.168.1.101
ansible_user=nlab-1

[all.children.managed_nodes.hosts.nlab-3]
ansible_host=192.168.1.103
ansible_user=nlab-3

[all.children.managed_nodes.hosts.nlab-4]
ansible_host=192.168.1.104
ansible_user=nlab-4
```

> **Bisa dijadikan Grup**
> ```
> all
> └── managed_nodes       ← parent group
>     ├── group_1         ← child group
>     │   ├── nlab-1
>     │   └── nlab-3
>     └── group_2         ← child group
>         └── nlab-4
> ```

```yaml
all:
  children:
    managed_nodes:
      children:
        group_1:
          hosts:
            nlab-1:
              ansible_host: 192.168.1.101
              ansible_user: nlab-1
            nlab-3:
              ansible_host: 192.168.1.103
              ansible_user: nlab-3
        group_2:
          hosts:
            nlab-4:
              ansible_host: 192.168.1.104
              ansible_user: nlab-4
```

```bash
# Semua node di managed_nodes (termasuk group_1 & group_2)
ansible managed_nodes -m ping

# Hanya group_1
ansible group_1 -m ping

# Hanya nlab-4
ansible nlab-4 -m ping

# Ping semua
ansible all -m ping

# lihat semua host terdaftar
ansible all --list-hosts
```

#### Tips Penamaan yang Lebih Praktis

Daripada `group_1` / `group_2`, lebih baik pakai nama fungsional:

| **Per role**      | **Per env**       | **Per lokasi**   |
|-------------------|-------------------|------------------|
| `web_servers`     | `staging`         | `dc_jakarta`     |
| `db_servers`      | `production`      | `dc_surabaya`    |

Ini penting karena Ansible variable inheritance mengikuti group hierarchy — `group_vars/web_servers/` otomatis apply ke semua host di grup itu.

> **Contoh variable inheritance**
>
> Ansible punya sistem hierarki variabel yang mengikuti struktur group di inventory. Variabel yang didefinisikan di parent group otomatis diwariskan ke semua child group dan host di dalamnya.
>
> ```
> all/
> ├── group_vars/
> │   ├── all.yml           ← berlaku untuk SEMUA host
> │   ├── managed_nodes.yml ← berlaku untuk semua di managed_nodes
> │   ├── web_servers.yml   ← berlaku untuk semua di web_servers
> │   └── db_servers.yml    ← berlaku untuk semua di db_servers
> └── host_vars/
>     ├── nlab-1.yml        ← hanya berlaku untuk nlab-1
>     └── nlab-3.yml        ← hanya berlaku untuk nlab-3
> ```

**Contoh Inventory:**
```yaml
all:
  children:
    managed_nodes:
      children:
        web_servers:
          hosts:
            nlab-1:
            nlab-3:
        db_servers:
          hosts:
            nlab-4:
```

**`group_vars/all.yml`**
```yaml
ntp_server: pool.ntp.org
timezone: Asia/Jakarta
```

**`group_vars/web_servers.yml`**
```yaml
nginx_port: 80
nginx_worker_processes: 4
```

**`group_vars/db_servers.yml`**
```yaml
mysql_port: 3306
mysql_max_connections: 200
```

**`host_vars/nlab-1.yml`**
```yaml
nginx_port: 8080   # ← override khusus nlab-1
```

**Hierarchy Precedence:**
```
all group_vars
    ↓
parent group_vars
    ↓
child group_vars
    ↓
host_vars          ← paling tinggi, selalu menang
```

---

### 2. Ansible Config (`ansible.cfg`)

File konfigurasi behavior Ansible. Dibaca secara hierarki — dari yang paling lokal:

```
./ansible.cfg          ← paling prioritas (project level)
~/.ansible.cfg
/etc/ansible/ansible.cfg
```

**Contoh:**
```ini
# ~/ansible-lab/ansible.cfg
[defaults]
inventory           = inventory/hosts.yml
private_key_file    = ~/.ssh/ansible_lab
host_key_checking   = False
stdout_callback     = yaml
callbacks_enabled   = timer, profile_tasks
vault_password_file = ~/.vault_pass
interpreter_python  = /usr/bin/python3.10

[privilege_escalation]
become        = True
become_method = sudo
become_user   = root

[ssh_connection]
pipelining = True
```

**Test:**
```bash
ansible --version
# baris "config file" harus menunjuk ke ./ansible.cfg
```

**Penjelasan:**

| Parameter | Penjelasan |
|-----------|------------|
| `inventory = inventory/hosts.yml` | Path ke inventory file. Relatif dari lokasi `ansible.cfg`. Jadi gak perlu `-i` tiap jalanin command. |
| `private_key_file = ~/.ssh/ansible_lab` | SSH private key yang dipakai untuk koneksi ke semua managed nodes. Pastikan public key-nya sudah di-copy ke tiap host (`~/.ssh/authorized_keys`). |
| `host_key_checking = False` | Disable verifikasi SSH host key. Berguna di lab/dev agar tidak prompt yes/no saat pertama connect. **Jangan pakai di production — rawan MITM attack.** |
| `stdout_callback = yaml` | Format output Ansible jadi YAML — lebih readable dibanding default `minimal`. Ada juga opsi `json`, `debug`, `dense`. |
| `callbacks_enabled = timer, profile_tasks` | Aktifkan dua callback plugin:<br>• `timer` → tampilkan total waktu eksekusi playbook<br>• `profile_tasks` → tampilkan durasi per task, berguna untuk identifikasi bottleneck |
| `vault_password_file = ~/.vault_pass` | Path ke file yang berisi password Ansible Vault. Isinya cuma plain text password. Jadi gak perlu `--vault-password-file` atau prompt tiap decrypt secret.<br>⚠️ **Pastikan permission file ini `600` dan tidak di-commit ke Git.** |
| `interpreter_python = /usr/bin/python3.10` | Paksa Ansible pakai Python versi spesifik di managed nodes. Mencegah warning/error akibat auto-detection yang salah pilih versi Python. |
| `become = True` | Aktifkan privilege escalation secara global |
| `become_method = sudo` | Gunakan `sudo` (opsi lain: `su`, `pbrun`, `doas`) |
| `become_user = root` | Escalate ke user `root` |

Artinya semua task otomatis jalan sebagai `root`, kecuali kamu override di level play/task dengan `become: false`.

| Parameter | Penjelasan |
|-----------|------------|
| `pipelining = True` | Ansible normalnya upload temporary Python script ke node lalu execute. Dengan pipelining, script dikirim langsung via SSH pipe tanpa nulis file sementara ke disk.<br><br>**Efeknya:**<br>• Eksekusi lebih cepat (signifikan di playbook besar)<br>• Mengurangi I/O di managed node<br><br>⚠️ **Requires `requiretty` di sudoers dinonaktifkan.** Tambahkan `Defaults !requiretty` di `/etc/sudoers` tiap managed node. |

> **Cara SSH tanpa PTY**
>
> ```yaml
> # bootstrap.yml — jalankan SEBELUM pipelining aktif
> - name: Bootstrap Ansible sudoers
>   hosts: managed_nodes
>   become: true
>   tasks:
>     - name: Drop nopty sudoers rule
>       copy:
>         dest: /etc/sudoers.d/ansible-nopty
>         content: "Defaults:{{ ansible_user }} !use_pty\n"
>         mode: "0440"
>         validate: "visudo -c -f %s"
> ```
>
> Jalankan sekali dengan `pipelining = False`, setelah itu baru enable `pipelining = True`.
> ```bash
> ansible-playbook bootstrap.yml
> ```

#### ⚠️ Risiko yang Perlu Diperhatikan

| Risiko | Mitigasi |
|--------|----------|
| User Ansible punya akses terlalu luas | Batasi sudo hanya ke command yang diperlukan |
| Private key bocor | Pakai passphrase + ssh-agent, rotasi key berkala |
| `host_key_checking = False` | Aktifkan di production, gunakan `known_hosts` |
| Vault password file di disk | Permission `600`, exclude dari backup umum |

#### Rekomendasi Tambahan — Least Privilege Sudoers

Daripada full sudo, batasi command yang boleh dijalankan Ansible:

```sudoers
# /etc/sudoers.d/ansible-nopty
Defaults:nlab-1 !use_pty
nlab-1 ALL=(root) NOPASSWD: /usr/bin/apt, /usr/bin/systemctl, /usr/bin/cp, /usr/bin/python3.10
```

Tapi ini tradeoff — makin restrictive, makin banyak maintenance kalau playbook berkembang. Untuk lab environment, full sudo masih acceptable.

---

### 3. Ansible Module

Module adalah unit kerja terkecil di Ansible — program yang dieksekusi di managed node untuk melakukan satu tugas spesifik. Setiap kali kamu tulis task di playbook, kamu sedang memanggil sebuah module.

**Analogi Simpel**
```
Ansible Playbook  =  resep masakan
Task              =  satu langkah di resep
Module            =  alat yang dipakai di langkah itu
```

```yaml
tasks:
  - name: Install nginx
    apt:              # ← ini MODULE
      name: nginx
      state: present

  - name: Start nginx
    systemd:          # ← ini MODULE
      name: nginx
      state: started

  - name: Copy config
    copy:             # ← ini MODULE
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
```

```bash
# Contoh ad-hoc pakai module langsung dari CLI

# module: ping
ansible all -m ping

# module: command
ansible all -m command -a "uptime"

# module: apt
ansible all -m apt -a "name=curl state=present" --become

# module: copy
ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts_backup"
```

Semua module Ansible dirancang **idempotent** — dijalankan 1x atau 10x, hasilnya sama, tidak ada efek samping.

```bash
ansible-doc -l
ansible-doc -l | grep docker
```

---

### 4. Ansible Playbook

Playbook adalah file YAML yang mendefinisikan otomasi — berisi instruksi lengkap: siapa yang dituju, apa yang dikerjakan, dan bagaimana urutannya.

Kalau module adalah alat, playbook adalah prosedur lengkapnya.

```
Playbook
└── Play 1                 ← satu play = satu grup host
    ├── vars
    ├── tasks
    │   ├── Task 1         ← memanggil module
    │   ├── Task 2
    │   └── Task 3
    ├── handlers
    └── roles
└── Play 2                 ← bisa multi-play dalam satu file
    └── tasks
```

```yaml
---
# Play 1 — Setup semua node
- name: Common setup
  hosts: managed_nodes
  become: true
  tasks:
    - name: Set timezone
      timezone:
        name: Asia/Jakarta

    - name: Install common packages
      apt:
        name:
          - curl
          - vim
          - htop
        state: present
        update_cache: true

# Play 2 — Khusus web server
- name: Setup web server
  hosts: web_servers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Copy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx        # ← trigger handler

  handlers:                        # ← hanya jalan kalau di-notify
    - name: Restart nginx
      systemd:
        name: nginx
        state: restarted

# Play 3 — Khusus database
- name: Setup database
  hosts: db_servers
  become: true
  tasks:
    - name: Install MySQL
      apt:
        name: mysql-server
        state: present
```

#### Handlers — task yang hanya jalan kalau dipanggil `notify`

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx    # ← hanya restart kalau config berubah

handlers:
  - name: Restart nginx
    systemd:
      name: nginx
      state: restarted
```

#### Conditionals — task berjalan hanya jika kondisi terpenuhi

```yaml
- name: Install nginx (hanya Ubuntu)
  apt:
    name: nginx
  when: ansible_os_family == "Debian"
```

#### Loops — iterasi task

```yaml
- name: Buat multiple user
  user:
    name: "{{ item }}"
    state: present
  loop:
    - deploy
    - appuser
    - monitor
```

#### Tags — jalankan task tertentu saja

```yaml
- name: Install nginx
  apt:
    name: nginx
  tags: [install, nginx]
```

#### Cara Jalankan

```bash
# Normal run
ansible-playbook site.yml

# Dry run — simulasi tanpa perubahan
ansible-playbook site.yml --check

# Lihat diff perubahan file
ansible-playbook site.yml --check --diff

# Limit ke host tertentu
ansible-playbook site.yml --limit nlab-1

# Limit ke tag tertentu
ansible-playbook site.yml --tags install
```