# HA Storage Server (DRBD)

### **Topologi Jaringan**

![image.png](images/image.png)

### **Konfigurasi Adapter Perangkat**

| **OS** | **Hostname** | **Ethernet/Adapter** | **Jenis Sambungan** |
| --- | --- | --- | --- |
| ubuntu server | master.skkkd.lab | eth1 | NAT |
|  |  | eth2 | Internal network |
|  |  | eth3 | Host Only |
| ubuntu server | backup.skkkd.lab | eth1 | NAT |
|  |  | eth2 | Internal network |
|  |  | eth3 | Host Only |
| linux client | client.skkkd.lab | eth1 | NAT |
|  |  | eth2 | Internal network |

### **Konfigurasi Jaringan**

| **Hostname** | master.skkkd.lab | backup.skkkd.lab |
| --- | --- | --- |
| **External Interface** | eth1 | eth1 |
| **External IP** | DHCP | DHCP |
| **Crossover interface** | eth2 | eth2 |
| **Crossover IP** | 10.10.10.1/28 | 10.10.10.2/28 |
| **Remote Interface** | eth3 | eth3 |
| **Remote IP** | 192.168.56.1/24 | 192.168.56.2/24 |
| **Virtual IP** | 10.10.10.14/28 |  |

### Teori Singkat

> DRBD (Distributed Replicated Block Device) merupakan salah satu teknologi untuk replikasi data antar server secara real-time. DRBD mirip dengan RAID-1 (mirroring), namun bukan lintas hard disk dalam satu server melainkan dilakukan antar dua server atau lebih. Semisal ada dua server yaitu server A dan server B, DRBD membuatnya tampak seperti dua server yang berbagi hard disk yang sama. Ketika Server A menulis data ke hard disk, DRBD secara otomatis mengirimkan salinan data tersebut ke Server B, secara real time. DRBD bekerja dengan mereplikasi data secara real-time antara dua atau lebih server untuk menciptakan sistem penyimpanan yang handal dan tahan terhadap kegagalan (high availability). DRBD beroperasi di level blok perangkat (block device), sehingga semua data yang ditulis ke perangkat DRBD di server utama (Primary) akan secara otomatis direplikasi ke server cadangan (Secondary) melalui jaringan. Hanya server yang berstatus Primary yang diperbolehkan membaca dan menulis ke perangkat DRBD, sedangkan server Secondary hanya menerima salinan data dan tidak bisa mengakses isi datanya secara langsung. Proses ini menjamin konsistensi data karena DRBD akan menunggu konfirmasi dari server Secondary bahwa data telah diterima sebelum menyelesaikan proses penulisan (pada mode sinkron seperti protocol C). Jika server Primary mengalami kegagalan, DRBD memungkinkan server Secondary untuk dinaikkan menjadi Primary, sehingga layanan tetap berjalan dengan data yang sama. Dengan pendekatan ini, DRBD menyediakan solusi replikasi data yang handal untuk membangun sistem storage yang tahan gangguan dan mendukung failover secara efisien.
> 

### **Menyiapkan VM**

Langakah awal, kita akan melakukan Clone VM dan menamainya dengan **master.skkkd.lab**. kemudian klik VM **master.skkkd.lab** selanjutnya  masuk ke ***settings*** dan pilih menu ***storage*** dan tambahkan hdd baru untuk replikasi penyimpanan.

![image.png](images/image%201.png)

Klik **create** dan tentukan ukuranya, misal 2 GB (sesuai kebutuhan) kemudian klik finish.

![image.png](images/image%202.png)

Klik pada HDD yang sudah dibuat dan kemudian klik **choose**

![image.png](images/image%203.png)

![image.png](images/image%204.png)

Langkah selanjutnya adalah menambahkan ethernet baru melalui menu ***network*** sehingga akan terbentuk seperti berikut ini:

![image.png](images/image%205.png)

Kemudian clone VM **master.skkkd.lab** menjadi **backup.skkkd.lab**, sehingga akan terbentu 2 VM seperti berikut

![image.png](images/image%206.png)

hidupkan VM **master.skkkd.lab** dan ketikkan perintah ***lsblk*** dan pastikan disk dengan ukuran 2G yang sudah dibuat tadi muncul.

```bash
$ lsblk
```

![image.png](images/image%207.png)

Selanjutnya yaitu lakukan konfigurasi hostname pada file ***/etc/hostname*** dan sesuaikan dengan hostname yang tertulis di tabel kebutuhan. Langkah berikutnya adalah melakukan konfigurasi IP untuk ethernet 1 dan ethernet 2 sesuai dengan tabel konfigurasi jaringan.

![image.png](images/image%208.png)

Selanjutnya ketikan perintah ***sudo netplan apply*** untuk menerapkan konfigurasi IP yang sudah dilakukan, dan pastikan **ethernet 1 (**enp0s3**) dan ethernet 2 (**enp0s8**)** sudah mendapatkan IP dengan mengeceknya menggunakan perintah ***ip a***

![image.png](images/image%209.png)

Berikutnya adalah  melakukan konfigurasi file host dengan cara  lakukan konfigurasi pada file ***/etc/hosts***

![image.png](images/image%2010.png)

Selanjutnya reboot vm **master.skkkd.lab.**

> Lakukan langkah yang sama untuk vm backup.skkkd.lab, sesuaikan ip dengan tabel konfigurasi jaringan dan pastikan kedua node sudah dapat terhubung antar node dan juga terhubung ke internet.
> 

### Installasi Package

Sebelum menuju ke langkah konfigurasi, langkah awal adalah menginstall paket yang dibutuhkan dengan menjalankan perintah berikut pada kedua node

```bash
$ sudo apt update
$ sudo apt install drbd-utils pacemaker corosync resource-agents-extra resource-agents-common resource-agents-base crmsh -y
```

jika pada saat installasi muncul tampilan postfix config seperti berikut, pilih ***No Configuration***

![image.png](images/image%2011.png)

### Konfigurasi DRBD

Setelah DRBD berhasil terinstall, selanjutnya kita perlu membuat file konfigurasi *resource* pada server **master.skkkd.lab***.* Buat file dengan ekstensi ***.res*** didalam folder ***/etc/drbd.d/***

```bash
$ sudo nano /etc/drbd.d/r0.res
```

kemudian lakukan konfigurasi berikut didalam file *resource* yang sudah dibuat.

```bash
resource r0 {
		protocol C;
		device /dev/drbd0;
		disk /dev/sdb;
		meta-disk internal;
		
		on master.skkkd.lab {
				address 10.10.10.1:7788;
		}
		
		on backup.skkkd.lab {
				address 10.10.10.2:7788;
		}
}
```

![image.png](images/image%2012.png)

Selanjutnya simpan konfigurasi tersebut. Lakukan konfigurasi yang sama untuk server **backup.skkkd.lab**. Langkah selanjutnya adalah membuat metadata DRBD pada resource **r0**  di server **master.skkkd.lab** dengan mengetikkan perintah berikut,

> Pastikan konfigurasi DRBD sudah benar
> 

```bash
$ sudo drbdadm create-md r0
```

![image.png](images/image%2013.png)

lakukan hal yang sama untuk vm **backup.skkkd.lab.**

kemudian aktifkan resource yang sudah dibuat dengan perintah berikut pada server **master.skkkd.lab**

$ sudo drbdadm up r0

> lakukan hal yang sama juga untuk vm **backup.skkkd.lab**
> 

untuk memastikan apakah kedua server sudah saling sinkron atau belum, cek status sinkronisasi drbd dengan menajalankan perintah berikut di salah satu server,

```bash
$ sudo drbdadm status
```

![image.png](images/image%2014.png)

Pada output di atas, dapat kita lihat bahwa kedua resource sudah berhasil terhubung, tetapi dalam keadaan **tidak konsisten** (*inconsistent state*) yang artinya penyimpanan pada kedua server belum saling sinkron satu sama lain. Supaya data dapat direplikasi dengan benar, kita perlu menempatkan resource ke dalam keadaan **konsisten**. Untuk mengatasi hal ini, ada dua acara yang dapat dilakukan :

> **Melakukan sinkronisasi penuh (full sync) pada perangkat (solusi 1)**
> 
> 
> Proses ini bisa saja memakan waktu lama, tergantung pada ukuran disk yang digunakan. Dengan cara memposisikan server master menjadi state primary, jalankan dua perintah berikut pada server **master.skkkd.lab**
> 
> ```bash
> $ sudo drbdadm primary --force r0
> $ sudo drbdadm -- --overwrite-data-of-peer primary r0
> ```
> 
> Jalankan perintah berikut di server master atau di server backup
> 
> ```bash
> $ sudo watch cat /proc/drbd
> ```
> 
> perintah ini digunakan untuk melihat/memonitoring progress proses sinkronisasi
> 
> ![image.png](images/image%2015.png)
> 
> **Melewati sinkronisasi awal (skip initial sync) (solusi 2)**
> 
> Opsi ini bisa dipilih jika kita yakin bahwa resource yang digunakan adalah **konfigurasi baru**, dengan **metadata baru** yang baru saja dibuat, dan **tidak ada data lama** di perangkat. Solusi kedua ini cocok untuk kasus seperti yang kita alami saat ini, karena kita memiliki *resource* yang baru terkonfigurasi dan bukan merupakan resource lama yang sudah terisi data-data. Jalankan perintah berikut pada node yang akan dijadikan sebagai ***Primary***
> 
> ```bash
> $ sudo drbdadm new-current-uuid --clear-bitmap r0/0
> ```
> 
> Karena dalam kasus ini kita sedang mengatur DRBD dari awal dengan metadata yang baru dibuat dan tanpa data lama di perangkat, maka kita akan memilih **solusi 2** (melewati sinkronisasi awal). Untuk melakukannya, jalankan perintah berikut di server **master.skkkd.lab** karena server master akan kita jadikan sebagai server primary :
> 
> ```bash
> $ sudo drbdadm new-current-uuid --clear-bitmap r0/0
> ```
> 
> kemudian lakukan monitoring progres sinkronisasi melalui server **backup.skkkd.lab**
> 
> ```bash
> $ cat /proc/drbd
> ```
> 
> ![image.png](images/image%2016.png)
> 
> Setelah proses sinkronisasi selesai, selanjutnya cek status sinkronisasi pada DRBD dengan menjalankan perintah berikut
> 
> ```bash
> $ sudo drbdadm status
> ```
> 
> ![image.png](images/image%2017.png)
> 
> Jika statusnya seperti diatas, artinya kedua server sudah saling sinkron. Namun role pada kedua server masih dalam keadaan ***Secondary,*** yang seharusnya server **master.skkkd.lab** memiliki role ***Primary*** dan **backup.skkkd.lab** memiliki role ***Secondary*.** Setelah kita berhasil menginisialisasi DRBD kita perlu mengubah server **master**.**skkkd.lab** menjadi state **Primary** serta membuat file system pada perangkat DRBD yang sudah kita tentukan diawal tadi (**r0**) dengan cara menjalankan perintah berikut pada server **master.skkkd.lab**
> 
> ```bash
> $ sudo drbdadm primary r0
> ```
> 
> ![image.png](images/image%2018.png)
> 
> Selanjutnya, di server **master.skkkd.lab**, format device ***/dev/drbd0*** dengan file sistem *EXT4*, jalankan perintah ini pada server **master.skkkd.lab**. Dengan memformat ***/dev/drbd0*** dengan sistem file **EXT4**, sehingga perangkat DRBD tersebut dapat digunakan untuk menyimpan data seperti partisi biasa, tapi dengan keunggulan replikasi data antar node.
> 
> ```bash
> $ sudo mkfs.ext4 /dev/drbd0
> ```
> 
> ![image.png](images/image%2019.png)
> 
> > Perintah ini akan memformat device ***/dev/drbd0*** dengan sistem file EXT4 sehingga
> > 
> > 
> > menjadikan ***/dev/drbd0*** siap digunakan untuk menyimpan data (misalnya bisa di-mount ke direktori tertentu). Kita hanya perlu melakukan hal ini di '**master.skkkd.lab**', dan bukan di '**backup.skkkd.lab**'. DRBD akan menangani replikasi tersebut.
> > 

Selanjutnya, buat direktori bernama ***/drbdpoint*** didalam direktori ***/mnt*** di kedua server yaitu **master.skkkd.lab** dan **backup.skkkd.lab :**

```bash
$ sudo mkdir /mnt/drbdpoint
```

Direktori ini digunakan untuk titik mount untuk perangkat DRBD. Setelah DRBD selesai mengonfigurasi replikasi antara node, sistem file yang dibuat pada perangkat DRBD di server **master.skkkd.lab** akan dimount ke direktori ini. Dengan hal ini, data yang disalin ke direktori ***/mnt/drbdpoint*** akan direplikasi secara otomatis ke node **backup.skkkd.lab** melalui DRBD. Konfigurasi DRBD awal sudah selesai, dan selanjutnya, kita akan mengonfigurasi Corosync di setiap node.

### Konfigurasi Corosync

- Pacemaker : Mengelola sumber daya klaster, seperti DRBD, dan mengatur failover atau pemulihan otomatis jika ada masalah pada salah satu node.
- Corosync : Menyediakan komunikasi dan sinkronisasi status antar node dalam klaster, memastikan bahwa Pacemaker mendapatkan informasi terbaru tentang keadaan (status) node-node yang ada.

Pastikan kedua paket tersebut sudah terinstall, kemudian  jalankan perintah berikut pada kedua server.

```bash
$ sudo systemctl enable corosync pacemaker
$ sudo systemctl start corosync pacemaker
```

Kemudian pada server **master.skkkd.lab**, masuk ke directory corosync dan backup file config defaultnya.

```bash
$ cd /etc/corosync
$ sudo cp corosync.conf corosync.conf.bak
```

Selanjutnya, edit file ***corosync.conf*** dengan *nano* dan hapus konfigurasi bawaan dengan cara 4 langkah berikut ini :

alt + \		⇒ 	ctrl + 6		⇒	alt + /	 ⇒	 ctrl + shift + k

setelah itu, kita tuliskan konfigurasi berikut ini ke dalam file corosync.conf :

```bash
totem {
		version: 2
		secauth: off
		cluster_name: ha_stg
		transport: knet
		rrp_mode: passive
}
nodelist {
	node {
		ring0_addr: 10.10.10.1
		nodeid: 1
		name: master.skkkd.lab
	}
	node {
		ring0_addr: 10.10.10.2
		nodeid: 2
		name: backup.skkkd.lab
	}
}
quorum {
		provider: corosync_votequorum
		two_node: 1
}
logging {
		to_syslog: yes
}
```

![image.png](images/image%2020.png)

> lakukan hal yang sama untuk node lainnya.
> 

Setelah corosync terkonfigurasi disemua node, langkah selanjutnya adalah merestart layanan/service corosync di semua node,

```bash
$ sudo systemctl restart corosync pacemaker
```

lihat status cluster dengan menggunakan crm_mon

```bash
$ sudo crm_mon
```

![image.png](images/image%2021.png)

Karena disini kita hanya menggunakan 2 node, maka kita perlu menseting agar pacemaker mengabaikan quorum. Jalankan perintah ini pada server **master.skkkd.lab**

```bash
$ sudo pcs property set no-quorum-policy=ignore
$ sudo pcs property set stonith-enabled=false
```

![image.png](images/image%2022.png)

### Konfigurasi Pacemaker untuk resource DRBD

Sekarang setelah kita berhasil menginisialisasi kluster, kita bisa mulai mengatur **Pacemaker** untuk mengelola ***resource*** (sumber daya). Gunakan perintah ***pcs cluster cib <nama_file>*** untuk membuat **file konfigurasi**. Di file inilah kita akan menulis atau menambahkan perubahan konfigurasi **sebelum** kita kirimkan / *push* ke kluster yang sedang berjalan. Simpelnya, kita membuat draf dulu di file, mengedit konfig disitu, lalu kemudian kirim ke sistem klusternya supaya aktif

*Resource* pertama yang akan kita konfigurasikan di Pacemaker adalah DRBD. Ketikan perintah – perintah berikut ini di node **master.skkkd.lab**, karena perintah-perintah ini akan diterapkan ke seluruh konfigurasi Pacemaker. Langkah-langkah ini akan mengambil versi konfigurasi kluster yang sedang berjalan (Cluster Information Base atau CIB), kemudian mengatur resource DRBD sebagai sebuah primitive dengan nama ***p_drbd_r0***, serta membuat clone yang dapat dipromosikan untuk resource DRBD tersebut. Setelah itu, konfigurasi akan diverifikasi dan perubahan akan diterapkan ke kluster.

```bash
$ sudo pcs cluster cib drbdconf 
$ sudo pcs -f drbdconf resource create p_drbd_r0 ocf:linbit:drbd drbd_resource=r0 op start interval=0s timeout=240s stop interval=0s timeout=100s monitor interval=31s timeout=20s role=Unpromoted monitor interval=29s timeout=20s role=Promoted
$ sudo pcs -f drbdconf resource promotable p_drbd_r0 promoted-max=1 promoted-node-max=1 clone-max=2 clone-node-max=1 notify=true
$ sudo pcs cluster cib-push drbdconf
```

![image.png](images/image%2023.png)

Sekarang, jika kita menjalankan perintah crm_mon untuk melihat status kluster, kita seharusnya bisa melihat bahwa **Pacemaker sudah mengelola perangkat DRBD kita**.

```bash
$ sudo crm_mon
```

![image.png](images/image%2024.png)

### Konfigurasi File System Primtive

Selanjutnya kita akan mengatur resource filesystem ***(p_fs_drbd0)*** agar hanya aktif jika DRBD ***(p_drbd_r0-clone)*** dalam kondisi **Primary.** Dengan keadaan DRBD berjalan, sekarang kita dapat melakukan konfigurasi file system kita dengan Pacemaker. Kita perlu melakukan konfigurasi **colocation** dan **order constraints** untuk memastikan bahwa file system dipasang *(mounted)* di node dimana DRBD menjadi *Promoted* *(Primary),* dan hanya ketika setelah DRBD dipromosikan terlebih dahulu. Jalankan perintah berikut pada server **master.skkkd.lab.**

```bash
$ sudo pcs -f drbdconf resource create p_fs_drbd0 ocf:heartbeat:Filesystem device=/dev/drbd0 directory=/mnt/drbdpoint fstype=ext4 options=noatime,nodiratime op start interval="0" timeout="60s" stop interval="0" timeout="60s" monitor interval="20" timeout="40s"
$ sudo pcs -f drbdconf constraint order promote p_drbd_r0-clone then start p_fs_drbd0
$ sudo pcs -f drbdconf constraint colocation add p_fs_drbd0 with p_drbd_r0-clone INFINITY with-rsc-role=Promoted
$ sudo pcs cluster cib-push drbdconf
$ sudo crm_mon
```

![image.png](images/image%2025.png)

Hasilnya akan seperti ini, akan ada reaource baru yaitu ***p_fs_drbd0***

![image.png](images/image%2026.png)

### Konfigurasi Layanan NFS (*NFS Exports*)

> lakukan konfigurasi berikut dikedua node
> 

Install nfs-kernel-server dan yang diperlukan

```bash
$ sudo apt install nfs-kernel-server rpcbind -y
$ sudo systemctl enable rpcbind --now
```

Karena sistem file harus di-*mount* (dipasang) secara lokal terlebih dahulu sebelum bisa dibagikan  oleh server NFS, maka kita perlu mengatur urutan dan keterkaitan antar *resources* (sumber daya) dengan benar. Artinya, layanan NFS hanya akan dijalankan di node yang sudah memiliki sistem file yang ter-*mount* (terpasang), dan hanya setelah proses *mounting* selesai. Setelah itu, barulah sumber daya ***exportfs*** bisa dijalankan di node yang sama. Pada contoh di bawah ini, perintah pcs akan otomatis membuat nama yang jelas dan bermakna untuk aturan urutan **(*order constraint*)** dan keterkaitan **(*colocation constraint*)**, dengan awalan order- dan colocation- yang ditambahkan ke nama sumber daya. Pertama-tama, kita perlu membuat ***primitive*** untuk layanan server NFS. Server NFS membutuhkan direktori khusus untuk menyimpan beberapa file penting. Direktori ini harus berada di dalam perangkat DRBD, karena file tersebut harus tersedia di tempat dan waktu yang tepat ketika server NFS dijalankan, agar proses ***failover*** berjalan lancar. Jalankan perintah berikut pada server **master.skkkd.lab.**

```bash
$ sudo pcs -f drbdconf resource create p_nfsserver ocf:heartbeat:nfsserver nfs_shared_infodir=/mnt/drbdpoint/nfs_shared_infodir nfs_ip=10.10.10.14 op start interval=0s timeout=40s stop interval=0s timeout=20s monitor interval=10s timeout=20s
$ sudo pcs -f drbdconf constraint colocation add p_nfsserver with p_fs_drbd0 INFINITY
$ sudo pcs -f drbdconf constraint order p_fs_drbd0 then p_nfsserver
$ sudo pcs cluster cib-push drbdconf
$ sudo crm_mon
```

![image.png](images/image%2027.png)

Hasilnya akan seperti ini, dimana akan ada resource baru bernama ***p_nfsserver***

![image.png](images/image%2028.png)

Dengan server NFS yang sudah berjalan, sekarang kita bisa membuat dan mengatur folder yang akan dibagikan (export), dari node mana pun yang sedang menjalankan resource p_nfsserver di dalam cluster (dalam kasus ini adalah master.skkkd.lab).

```bash
$ sudo mkdir -p /mnt/drbdpoint/exports/dir
$ sudo chown nobody:nogroup /mnt/drbdpoint/exports/dir
$ sudo pcs -f drbdconf resource create p_exportfs_dir ocf:heartbeat:exportfs clientspec=10.10.10.0/28 directory=/mnt/drbdpoint/exports/dir fsid=1 unlock_on_stop=1 options=rw,sync op start interval=0s timeout=40s stop interval=0s timeout=120s monitor interval=10s timeout=20s
$ sudo pcs -f drbdconf constraint order p_nfsserver then p_exportfs_dir
$ sudo pcs -f drbdconf constraint colocation add p_exportfs_dir with p_nfsserver INFINITY
$ sudo pcs cluster cib-push drbdconf
```

![image.png](images/image%2029.png)

Maka akan muncul resource baru bernama ***p_exportfs_dir***

![image.png](images/image%2030.png)

Sekarang, gunakan perintah showmount **dari sistem klien** untuk melihat direktori yang dibagikan (*exported*) oleh node utama (*Primary*) saat ini (dalam contoh ini adalah **master.skkkd.lab**).

> sudo showmount -e <ip node primary saat ini>
> 

Pastikan client sudah bisa terhubung ke server, uji coba dengan ping ke ip server master.skkkd.lab atau backup.skkkd.lab

```bash
$ sudo showmount -e 10.10.10.1
```

![image.png](images/image%2031.png)

### Konfigurasi IP Virtual

Virtual IP (VIP) akan memberikan akses yang konsisten bagi klien ke layanan NFS, meskipun salah satu node dalam cluster mengalami kegagalan. Untuk merealisasikan hal ini, kita perlu membuat aturan ***colocation*** (penempatan bersama), agar resource VIP selalu aktif di node yang sama dengan NFS export yang sedang berjalan. Jadi nantinya client cukup perlu mengetahui VIP-nya saja tanpa harus memikirkan IP dari node ***Primary**.* Lakukan konfigurasi berikut pada server **master.skkkd.lab**

```bash
$ sudo pcs -f drbdconf resource create p_virtip_dir ocf:heartbeat:IPaddr2 ip=10.10.10.14 cidr_netmask=28 op monitor interval=20s timeout=20s start interval=0s timeout=20s stop interval=0s timeout=20s
$ sudo pcs -f drbdconf constraint order p_exportfs_dir then p_virtip_dir
$ sudo pcs -f drbdconf constraint colocation add p_virtip_dir with p_exportfs_dir INFINITY
$ sudo pcs cluster cib-push drbdconf
```

![image.png](images/image%2032.png)

Maka hasilnya akan seperti berikut ini, yaitu akan ada resource bernama ***p_virtual_ip_dir***

![image.png](images/image%2033.png)

### Pengujian

Sebelum melakukan pengujian, install nfs common disisi client dengan cara mengetikan

```bash
$ sudo apt install nfs-common
```

Uji koneksi dari client ke IP Virtual dengan melakukan ping terlebih dahulu dari sisi client ke virtual ip, jika hasilnya tidak RTO artinya konfigurasi virtual ip sudah berhasil dan client bisa terhubung ke IP virtual tadi. Selanjutnya, lihat berkas yang diekspor dengan menggunakan perintah showmount, namun kali ini IP yang digunakan bukan lagi IP ***node Primary**,* melaikan IP virtual.

```bash
$ sudo showmount -e 10.10.10.14
```

![image.png](images/image%2034.png)

Pertama-tama, kita perlu membuat folder di sisi client. Folder ini nantinya sebagai tempat untuk menampung file system yang akan kita akses dari server. Setelah foldernya siap, baru kita lakukan proses ***mount*** yaitu menyambungkan direktori dari server ke folder lokal tadi. Dengan begitu, isi dari direktori di server bisa langsung diakses dari perangkat client seolah-olah itu bagian dari sistem lokal

```bash
$ sudo mkdir /mnt/nfs_mount
$ sudo mount 10.10.10.14:/mnt/drbdpoint/exports/dir /mnt/nfs_mount
```

![image.png](images/image%2035.png)

Setelah kita membuat direktori ***mounting point***  dan berhasil melakukan mounting, selanjutnya kita akan melakukan ujicoba dengan membuat file baru di sisi client dengan nama ***client.txt***

![image.png](images/image%2036.png)

Kemudian cek apakah file tersebut berhasil masuk ke server node primary, yaitu **master.skkkd.lab**

![image.png](images/image%2037.png)

Berikutnya kita akan mencoba dari sisi server dimana kita akan membuat file di sisi server primary.

![image.png](images/image%2038.png)

Selanjutnya, periksa di sisi client untuk memastikan bahwa file yang dibuat di sisi server juga muncul di client. Hal ini menunjukkan bahwa client dan server telah tersinkronisasi dengan baik.

![image.png](images/image%2039.png)

### Uji Coba Failover

Setelah berhasil melakukan pengujian sinkronasi file, langkah berikutnya adalah melakukan uji coba failover dengan *reboot / poweroff* server master dengan mengetikan perintah sudo poweroff dan pantau cluster dengan sudo crm_mon di node backup.

![image.png](images/image%2040.png)

![image.png](images/image%2041.png)

Dapat kita lihat gambar diatas, bahwa ***resource*** berhasil berpindah dari yang semula berada di server **master** (gambar 4. 41) beralih ke server **backup** (gambar 4. 42).  Kemudian ujicoba pada client apakah resource masih bisa diakses atau tidak. Biasanya untuk mengakses isi file membutuhkan waktu karena saat peralihan node akan terjadi downtime beberapa detik.

![image.png](images/image%2042.png)

Selanjutnya lakukan pengujian kembali dengan cara edit isi file ***server.txt*** melalui node **backup.skkkd.lab**

![image.png](images/image%2043.png)

Kemudian periksa kembali isi file ***server.txt*** dari sisi client, pastikan isinya sesuai / sama dengan apa yang sudah ditulis pada sisi server backup.

![image.png](images/image%2044.png)

Selama proses pengujian, bisa ktia perhatikan bahwa file yang ditulis di *node backup* bisa muncul dan diakses tanpa masalah. Itu berarti data berhasil tersinkronisasi dengan baik. Saat dilakukan failover, layanan berpindah dari satu node ke node lainnya, dan disini kita masih bisa membuka data seperti biasa dari sisi *client* tanpa ada error ataupun kehilangan data. Ini menunjukkan bahwa sistem bisa beralih otomatis tanpa memengaruhi akses data. Dari awal sampai akhir, mulai dari konfigurasi replikasi, mount file system, sampai uji coba failover, semua berjalan lancar, maka dapat kita simpulkan konfigurasi dasar sistem high availability storage server ini berhasil dan berjalan seperti yang diharapkan.