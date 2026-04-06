- name: Belajar Variables
  hosts: managed_nodes
  become: true

  # Cara 1: vars langsung di playbook
  vars:
    app_user: "deployuser"
    app_dir: "/opt/myapp"
    packages:
      - git
      - unzip
      - jq

  tasks:
    - name: Buat direktori app
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Install packages dari variabel list
      apt:
        name: "{{ packages }}"
        state: present

    - name: Buat user dari variabel
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: true

    # Cara 2: register — simpan output task ke variabel
    - name: Cek disk usage
      command: df -h /
      register: disk_info
      changed_when: false

    - name: Tampilkan disk usage
      debug:
        msg: "{{ disk_info.stdout_lines }}"

    # Cara 3: ansible facts — variabel otomatis dari node
    - name: Tampilkan info dari facts
      debug:
        msg: >
          Host: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          RAM: {{ ansible_memtotal_mb }} MB