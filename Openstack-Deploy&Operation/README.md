Panduan Lengkap Deployment OpenStack (Kolla-Ansible) & Troubleshooting
Jalankan seluruh langkah di bawah ini langsung dari Node 1: Controller (opstcont - 172.16.10.100) menggunakan user yang memiliki akses sudo (atau langsung sebagai root).
Langkah 1: Instalasi Ansible & Kolla-Ansible di Controller (opstcont)
Kita perlu memasang Python, Ansible, dan dependensi Kolla-Ansible terlebih dahulu di node Controller.
# 1. Update package manager
sudo apt update && sudo apt upgrade -y

# 2. Pasang Python3 pip dan virtual environment (Disesuaikan untuk Ubuntu 24.04)
sudo apt install python3-pip python3-venv -y

# 3. Buat Virtual Environment khusus untuk Kolla-Ansible agar tidak bentrok dengan OS
python3 -m venv ~/kolla-venv
source ~/kolla-venv/bin/activate

# 4. Upgrade pip dan pasang versi Ansible-core yang didukung (2.15 atau 2.16)
pip install -U pip
pip install "ansible-core>=2.15,<2.17"

# 5. Pasang Kolla-Ansible via PyPI (Menggunakan Versi 18.x.x untuk rilis OpenStack 2024.1 / Caracal)
pip install "kolla-ansible>=18.0.0,<19.0.0"

# 5a. Bypass Error Git Clone Galaxy dengan Build Koleksi secara Manual (Sangat Direkomendasikan)
# Clone repositori koleksi Kolla secara utuh
git clone [https://opendev.org/openstack/ansible-collection-kolla.git](https://opendev.org/openstack/ansible-collection-kolla.git) ~/ansible-collection-kolla
cd ~/ansible-collection-kolla

# Gunakan branch master sebagai fallback yang stabil dan selalu tersedia di remote
git checkout master

# Bangun paket koleksi menjadi tarball .tar.gz lokal
ansible-galaxy collection build

# Install file tarball yang berhasil dibuat ke virtual environment
ansible-galaxy collection install openstack-kolla-*.tar.gz

# Pasang dependensi modul sistem yang mutlak dibutuhkan oleh peran baremetal Kolla, loadbalancer, prechecks, dan deploy
ansible-galaxy collection install ansible.posix community.general ansible.utils community.docker containers.podman

# Install pustaka python netaddr (diperlukan untuk filter ipaddr Ansible)
pip install netaddr

# Kembali ke folder home
cd ~

# 6. Buat folder konfigurasi Kolla di sistem
sudo mkdir -p /etc/kolla
sudo chown -R $USER:$USER /etc/kolla

# 7. Salin file template bawaan Kolla ke folder /etc/kolla
cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/multinode .


#Terus jalanin ini didalem Computenode
sudo apt update
sudo apt install openvswitch-switch -y
apt install -y tgt
systemctl enable --now tgt
systemctl status tgt 

apt install -y open-iscsi
systemctl enable --now iscsid
systemctl status iscsid open-iscsi --no-pager 

Langkah 2: Edit File Globals Configuration (/etc/kolla/globals.yml)
File /etc/kolla/globals.yml adalah file konfigurasi utama yang mengatur seluruh sistem OpenStack lu.
Buka filenya menggunakan editor nano:
nano /etc/kolla/globals.yml


Hapus isinya (atau timpa) dengan konfigurasi yang sudah disesuaikan khusus untuk lingkungan kluster 3 node lu berikut ini.
Catatan: Asumsi nama interface management (NIC 1) adalah eth0 dan interface eksternal (NIC 2) adalah eth1. Sesuaikan dengan nama interface asli di server lu (cek pakai perintah ip a).
# =====================================================================
# KONFIGURASI UTAMA KOLLA-ANSIBLE UNTUK KLUSTER 3-NODE BEINT.LOCAL
# =====================================================================

# 1. Distro & Versi OpenStack (Menggunakan Caracal untuk Ubuntu 24.04)
kolla_base_distro: "ubuntu"
openstack_release: "2024.1"

# 2. Metode Deployment (Containerized)
kolla_install_type: "binary"

# 3. Virtual IP (VIP) Internal OpenStack
# Kita butuh 1 IP kosong di subnet 172.16.10.0/22 sebagai gerbang API & Dashboard Horizon.
# Pastikan IP ini TIDAK dipakai oleh node fisik mana pun. Kita gunakan .99 sebagai contoh.
kolla_internal_vip_address: "172.16.10.99"

# 4. Konfigurasi Network Interface
# NIC 1: Management Network (Tempat IP 172.16.10.x berada)
network_interface: "eth0"

# NIC 2: Provider / External Network (Dicolok ke internet/VLAN luar, tanpa IP di OS host)
neutron_plugin_agent: "openvswitch" 
neutron_external_interface: "ens192"
neutron_bridge_name: "br-ex"
enable_neutron_provider_networks: "yes"
enable_neutron_dvr: "yes" 


# Bypass otomatisasi /etc/hosts Kolla (Mencegah Error templating override_var)
customize_etc_hosts: "no"

# 5. Aktifkan Layanan Core OpenStack
enable_horizon: "yes"      # Dashboard Web UI
enable_glance: "yes"       # Image Service (Penyimpan OS template VM)
enable_cinder: "yes"       # Block Storage (Volume disk tambahan VM)
enable_neutron_dvr: "yes"  # Distributed Virtual Routing (Network ditaruh di tiap compute)

# 6. Konfigurasi Database & Broker
# Karena kita hanya punya 1 Controller, matikan proteksi clustering multi-controller
enable_mariadb_backup: "no"

# 7. Konfigurasi Cinder Backend (Storage VM)
# Kita gunakan LVM di compute node untuk backend penyimpanan disk VM standar
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"

# 8. Integrasi Virtualisasi (KVM)
# Karena kita nested di atas VMware, wajib menggunakan KVM hypervisor
nova_compute_virt_type: "kvm"


Gunakan tombol Ctrl + O -> Enter untuk menyimpan, lalu Ctrl + X untuk keluar dari nano.
Langkah 2a: Edit File Inventory (~/multinode)
Buka berkas multinode yang baru saja kita salin ke folder home (~) Anda sebelumnya:
nano ~/multinode


Hapus seluruh isi template bawaan di dalamnya, lalu timpa dengan pemetaan IP dan peran kluster 3-node Anda yang sebenarnya di bawah ini:
# =====================================================================
# INVENTORY KOLLA-ANSIBLE 3-NODE CLUSTER (BEINT.LOCAL)
# =====================================================================

# --- GRUP UTAMA (PEMBAGIAN IP FISIK NODE) ---

[control]
172.16.10.100  # opstcont.beint.local

[network]
172.16.10.101  # opstcomp1.beint.local
172.16.10.102  # opstcomp2.beint.local

[compute]
172.16.10.101  # opstcomp1.beint.local
172.16.10.102  # opstcomp2.beint.local

[monitoring]
172.16.10.100  # opstcont.beint.local

[storage]
172.16.10.101  # opstcomp1.beint.local
172.16.10.102  # opstcomp2.beint.local

[deployment]
localhost ansible_connection=local


# --- GRUP TURUNAN (AUTOMATIC MAPPING UNTUK LAYANAN OPENSTACK) ---

[baremetal:children]
control
network
compute
storage

[tls-backend:children]
control

# Common Services
[common:children]
control
network
compute
storage

[cron:children]
common

[kolla-toolbox:children]
common

[fluentd:children]
common

# Logging (Mengatasi Error Creating log volume)
[kolla-logs:children]
common

# Loadbalancer (HAProxy)
[loadbalancer:children]
network

# Database & Broker
[mariadb:children]
control

[rabbitmq:children]
control

[memcached:children]
control

# Identity (Keystone)
[keystone:children]
control

# Image (Glance)
[glance:children]
glance-api

[glance-api:children]
control

# Placement
[placement:children]
placement-api

[placement-api:children]
control

# Compute (Nova API, Agents & Proxies)
[nova:children]
nova-api
nova-conductor
nova-super-conductor
nova-scheduler
nova-novncproxy
nova-spicehtml5proxy
nova-serialproxy
nova-compute
nova-compute-ironic
nova-libvirt
nova-ssh

[nova-api:children]
control

[nova-conductor:children]
control

[nova-super-conductor:children]
control

[nova-novncproxy:children]
control

[nova-spicehtml5proxy:children]
control

[nova-serialproxy:children]
control

[nova-scheduler:children]
control

[nova-compute:children]
compute

[nova-compute-ironic:children]
compute

[nova-libvirt:children]
compute

[nova-ssh:children]
compute

# Network (Neutron Server & Agents)
[neutron:children]
neutron-server
neutron-dhcp-agent
neutron-l3-agent
neutron-metadata-agent
neutron-openvswitch-agent
neutron-bgp-dragent
neutron-infoblox-ipam-agent
neutron-metering-agent
neutron-sriov-agent

[neutron-server:children]
control

[neutron-dhcp-agent:children]
network

[neutron-l3-agent:children]
network

[neutron-metadata-agent:children]
network

[neutron-openvswitch-agent:children]
compute
network

[neutron-bgp-dragent:children]
network

[neutron-infoblox-ipam-agent:children]
network

[neutron-metering-agent:children]
network

[neutron-sriov-agent:children]
compute

# Block Storage (Cinder API, Conductor, Volume & Backup)
[cinder:children]
cinder-api
cinder-conductor
cinder-scheduler
cinder-volume
cinder-backup

[cinder-api:children]
control

[cinder-conductor:children]
control

[cinder-scheduler:children]
control

[cinder-volume:children]
storage

[cinder-backup:children]
storage

# Dashboard (Horizon)
[horizon:children]
control

# Orchestration (Heat)
[heat:children]
heat-api
heat-api-cfn
heat-engine

[heat-api:children]
control

[heat-api-cfn:children]
control

[heat-engine:children]
control

# Grafana & Prometheus (Monitoring)
[grafana:children]
monitoring

[prometheus:children]
monitoring

[prometheus-node-exporter:children]
control
network
compute
storage


# --- GRUP WAJIB & BACKEND LAINNYA (STUB UNTUK MENGHINDARI ERROR DEFINISI PRECHECK) ---
[bifrost]

[ironic]

[neutron-linuxbridge-agent]

[neutron-ovn-metadata-agent]

[neutron-ovn-agent]

[neutron-mlnx-agent]

[neutron-eswitchd]

[ironic-neutron-agent]


# --- KONFIGURASI AKSES ANSIBLE ---
[all:vars]
ansible_user=root
ansible_become=True
ansible_become_method=sudo


Gunakan tombol Ctrl + O -> Enter untuk menyimpan, lalu Ctrl + X untuk keluar.
Langkah 3: Membuat Volume Group LVM untuk Storage (Cinder)
Karena kita mengaktifkan Cinder LVM (enable_cinder_backend_lvm: "yes"), kita harus menyiapkan Volume Group bernama cinder-volumes di Node 2 (opstcomp1) dan Node 3 (opstcomp2).
Lakukan ini di opstcomp1 dan opstcomp2 menggunakan disk kosong kedua (misal: /dev/sdb):
# 1. Pastikan lvm2 sudah terpasang
sudo apt update && sudo apt install lvm2 -y

# 2. Buat Physical Volume pada disk kedua kosong lu (misal /dev/sdb)
sudo pvcreate /dev/sdb

# 3. Buat Volume Group dengan nama persis 'cinder-volumes'
sudo vgcreate cinder-volumes /dev/sdb


Langkah 4: Generate Password Otomatis
Kolla-Ansible menyediakan skrip untuk membuat kata sandi acak (seperti password DB, password admin Horizon, dll) yang akan disimpan secara aman di /etc/kolla/passwords.yml.
Jalankan perintah ini di Controller (opstcont):
# Pastikan virtual environment aktif terlebih dahulu
source ~/kolla-venv/bin/activate

# Jalankan skrip pembuat password
kolla-genpwd


Jika lu ingin melihat atau mengubah password admin untuk login Dashboard Horizon nanti, lu bisa melihatnya di file tersebut:
grep "keystone_admin_password" /etc/kolla/passwords.yml


Langkah 5: Persiapan Host (Bootstrap Servers)
Ansible sekarang akan masuk ke ketiga node (opstcont, opstcomp1, opstcomp2) untuk mengonfigurasi host OS, memasang Docker Engine, mengatur kernel parameters, dan menyiapkan folder-folder yang dibutuhkan.
Jalankan perintah ini di Controller (opstcont):
# Pastikan multinode inventory yang lu edit sebelumnya sudah berada di folder aktif lu.
# Jalankan proses bootstrap server
kolla-ansible -i ~/multinode bootstrap-servers


Langkah 6: Pengujian Kelayakan Hardware (Prechecks)
Sebelum men-deploy secara utuh (yang memakan waktu cukup lama), kita harus melakukan validasi apakah spesifikasi hardware, port-port jaringan, dan konfigurasi IP kita sudah sesuai kriteria OpenStack.
Jalankan perintah ini di Controller (opstcont):
kolla-ansible -i ~/multinode prechecks


Langkah 7: Eksekusi Deployment Akhir (Deploy!)
Ini adalah proses utama di mana Ansible akan menarik (pull) seluruh Docker container image OpenStack dari registry dan menyalakan layanannya di ketiga node secara otomatis.
Jalankan perintah ini di Controller (opstcont):
kolla-ansible -i ~/multinode deploy


Langkah 8: Post-Deployment (Membuat File Akses CLI)
Setelah deploy sukses tanpa error, buat file kredensial agar lu bisa mengendalikan OpenStack lu lewat terminal CLI.
# Membuat file admin-openrc.sh
kolla-ansible post-deploy

# Untuk mulai menggunakan CLI OpenStack, load filenya ke session terminal lu:
. /etc/kolla/admin-openrc.sh

# Pasang OpenStack Client CLI tools
pip install python-openstackclient

# Uji coba apakah OpenStack sudah merespon:
openstack service list


Sekarang OpenStack lu sudah siap! Lu bisa membuka browser laptop lu dan mengakses Dashboard Horizon lewat IP VIP yang sudah diset tadi di alamat: http://172.16.10.99.
Solusi Troubleshooting & Penanganan Error Umum
1. Mengatasi SSH Delay / Lambat saat Menghubungkan Antar Node
Jika di-ping instan namun ketika di-SSH mengalami jeda/delay lambat sekitar 10-30 detik sebelum meminta password.
Langkah Perbaikan (Jalankan di Node Compute 101 & 102):
Buka file konfigurasi SSH Daemon:
sudo nano /etc/ssh/sshd_config


Tambahkan/ubah baris berikut di bagian terbawah file:
UseDNS no
GSSAPIAuthentication no


Simpan berkas (Ctrl + O -> Enter -> Ctrl + X).
Restart SSH service:
sudo systemctl restart ssh


2. Mengatasi I/O Timeout Quay.io saat Deployment (DI SEMUA NODE)
Error i/o timeout (seperti failed to resolve reference... dial tcp: lookup quay.io) terjadi karena server DNS bawaan Linux kewalahan atau tidak merespons cepat saat Docker daemon berusaha mengunduh container dari server Quay.io.
Karena Kolla-Ansible mengunduh image ke seluruh node, lu harus memastikan DNS stabil di semua node yang mengalami error.
Langkah Solusi Jitu:
Ganti DNS ke Google di Node yang Bermasalah (Sangat Disarankan di Ketiga Node)
Buka SSH langsung ke node yang dilaporkan gagal di log Ansible (contoh kali ini 172.16.10.101 / Node 2), lalu jalankan:
sudo nano /etc/resolv.conf

Ubah atau tambahkan baris ini di posisi paling atas:
nameserver 8.8.8.8
nameserver 1.1.1.1

Simpan file (Ctrl+O -> Enter -> Ctrl+X). Lakukan hal yang sama ke 172.16.10.102 biar nggak nyusul error serupa.
Tarik (Pull) Image Secara Manual di Node Tersebut
Melihat log lu, Ansible gagal menarik image kolla-toolbox. Selagi lu SSH di 172.16.10.101, pancing Docker-nya pakai perintah manual ini:
docker pull quay.io/openstack.kolla/kolla-toolbox:2024.1-ubuntu-jammy

Tunggu sampai 100% Downloaded.
Jalankan Ulang Perintah Deploy dari Controller
Kembali ke layar SSH Controller (opstcont), lalu jalankan kembali senjatanya:
kolla-ansible -i ~/multinode deploy

Ansible akan langsung lanjut memproses node 2 tanpa rintangan!
