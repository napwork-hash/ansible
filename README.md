Tentu, ini adalah versi Markdown yang telah saya rapikan strukturnya. Saya menggunakan teknik *visual hierarchy* agar informasi teknis yang padat ini lebih mudah dipindai (scannable) dan dipahami tanpa mengubah konten aslinya.

---

# 🚀 Ansible Core Concepts — Hands-on Lab

## 📂 Struktur Folder
```text
C:.
├── .codex
├── .vault_pass
├── ansible.cfg
├── Vault.md
├── inventory/
│   ├── hosts.yml
│   └── host_vars/
│       ├── nlab-1.yaml
│       ├── nlab-1.yaml.bak
│       ├── nlab-3.yaml
│       └── nlab-4.yaml
└── playbooks/
    ├── debug_become.yaml
    ├── htop.yaml
    ├── variables.md
    └── variables.yaml
```

---

## 🔑 SSH & Konfigurasi IP
Pastikan **IP Address Static** (bisa menggunakan `netplan`). Setiap VM Worker wajib memiliki SSH Key dari Master Controller.

### Workflow Setup Key
1.  **Generate Key (Master)**
    ```bash
    ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/ansible_lab -N ""
    ```
2.  **Pasang Key ke Worker**
    ```bash
    ssh-copy-id -i ~/.ssh/ansible_lab.pub nlab-1@192.168.1.101
    ```
3.  **Uji Koneksi**
    ```bash
    ssh -i ~/.ssh/ansible_lab nlab-1@192.168.1.101
    ```

---

## 🏗️ Core Concepts

### 1. Ansible Inventory
Daftar *managed nodes* dalam format YAML atau INI.



#### **Format YAML**
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

#### **Struktur Grup (Hierarki)**
```text
all
└── managed_nodes         ← parent group
    ├── group_1           ← child group (nlab-1, nlab-3)
    └── group_2           ← child group (nlab-4)
```

#### **Tips Penamaan Praktis**
Gunakan nama fungsional agar *variable inheritance* lebih efektif:
* **Per Role:** `web_servers`, `db_servers`
* **Per Env:** `staging`, `production`
* **Per Lokasi:** `dc_jakarta`, `dc_surabaya`

---

### 2. Variable Inheritance (Pewarisan Variabel)
Variabel mengalir dari grup umum ke spesifik.



**Urutan Prioritas (Bawah ke Atas):**
1.  `all` group_vars (Paling rendah)
2.  Parent group_vars
3.  Child group_vars
4.  **host_vars** (Paling tinggi, selalu menang)

**Contoh Kasus:**
Jika `group_vars/all.yml` mengatur `timezone: UTC` tapi `host_vars/nlab-1.yml` mengatur `timezone: Asia/Jakarta`, maka `nlab-1` akan menggunakan **Asia/Jakarta**.

---

### 3. Ansible Config (`ansible.cfg`)
Dibaca berdasarkan urutan prioritas: `./ansible.cfg` (Project) > `~/.ansible.cfg` > `/etc/ansible/ansible.cfg`.

#### **Konfigurasi Utama**
* `inventory`: Path ke file inventory (menghindari penggunaan `-i`).
* `private_key_file`: SSH private key untuk koneksi otomatis.
* `host_key_checking = False`: Matikan verifikasi SSH host key (hanya untuk Lab/Dev).
* `stdout_callback = yaml`: Membuat output lebih rapi dan terbaca.
* `vault_password_file`: Lokasi file password Vault (Gunakan permission 600!).
* `pipelining = True`: Eksekusi lebih cepat dengan mengirim script via SSH pipe (Membutuhkan penyesuaian sudoers).

#### **🛡️ Tabel Risiko & Mitigasi**
| Risiko | Mitigasi |
| :--- | :--- |
| Akses sudo terlalu luas | Gunakan **Least Privilege Sudoers** |
| Private key bocor | Pakai passphrase + ssh-agent |
| `host_key_checking = False` | Aktifkan di production (cegah MITM) |
| Vault password di disk | Set permission 600 & exclude dari backup |

---

### 4. Ansible Module
Unit kerja terkecil yang bersifat **Idempotent** (dijalankan berulang kali tetap menghasilkan status yang sama).

* **Analogi:** Playbook (Resep) → Task (Langkah) → **Module (Alat)**.

**Contoh Ad-hoc Command:**
```bash
ansible all -m ping                                     # Cek koneksi
ansible all -m command -a "uptime"                     # Jalankan perintah shell
ansible all -m apt -a "name=curl state=present" --become # Install paket
```

---

### 5. Ansible Playbook
File YAML yang mendefinisikan prosedur otomasi lengkap.



#### **Komponen Utama**
* **Handlers:** Task yang hanya jalan jika di-`notify` oleh task lain (contoh: restart nginx hanya jika config berubah).
* **Conditionals (`when`):** Jalankan task hanya jika syarat terpenuhi (contoh: `ansible_os_family == "Debian"`).
* **Loops (`loop`):** Iterasi untuk tugas berulang (contoh: membuat banyak user sekaligus).
* **Tags:** Menjalankan bagian spesifik dari playbook menggunakan opsi `--tags`.

#### **Perintah Eksekusi**
```bash
ansible-playbook site.yml              # Jalankan normal
ansible-playbook site.yml --check      # Dry run (simulasi)
ansible-playbook site.yml --limit host # Batasi ke host tertentu
```

---

### 💡 Rekomendasi Sudoers (Least Privilege)
Untuk keamanan lebih baik, batasi perintah sudo yang bisa dijalankan Ansible:
```ini
# /etc/sudoers.d/ansible-nopty
Defaults:nlab-1 !use_pty
nlab-1 ALL=(root) NOPASSWD: /usr/bin/apt, /usr/bin/systemctl, /usr/bin/cp, /usr/bin/python3.10
```

---