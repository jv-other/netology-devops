# 3.5. Файловые системы

#### 1. Узнайте о [sparse](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB) (разряженных) файлах.

---

Разрежённый файл (sparse) — файл, в котором последовательности нулевых байтов заменены на информацию об этих последовательностях (список дыр). 

#### 2. Могут ли файлы, являющиеся жесткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?

---

Нет, разрешения как и другие атрибуты структуры inode хранятся на диске. Поскольку жесткая ссылка только связывает индексный дескриптор с каталогом и даёт ему имя, все данные inode для всех ссылок будут одинаковые.

#### 3. Сделайте `vagrant destroy` на имеющийся инстанс Ubuntu. Замените содержимое Vagrantfile следующим:

```bash
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.provider :virtualbox do |vb|
    lvm_experiments_disk0_path = "/tmp/lvm_experiments_disk0.vmdk"
    lvm_experiments_disk1_path = "/tmp/lvm_experiments_disk1.vmdk"
    vb.customize ['createmedium', '--filename', lvm_experiments_disk0_path, '--size', 2560]
    vb.customize ['createmedium', '--filename', lvm_experiments_disk1_path, '--size', 2560]
    vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', lvm_experiments_disk0_path]
    vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', lvm_experiments_disk1_path]
  end
end
```

Данная конфигурация создаст новую виртуальную машину с двумя дополнительными неразмеченными дисками по 2.5 Гб.

---

```bash
lsblk | grep disk

>>> sda                         8:0    0   64G  0 disk
>>> sdb                         8:16   0  2.5G  0 disk
>>> sdc                         8:32   0  2.5G  0 disk
```

#### 4. Используя `fdisk`, разбейте первый диск на 2 раздела: 2 Гб, оставшееся пространство.

---

```bash
sudo fdisk /dev/sdb

>>> Command (m for help): n
>>> Select (default p): p
>>> Partition number (1-4, default 1): 1
>>> First sector (2048-5242879, default 2048):
>>> Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-5242879, default 5242879): +2G
>>> Created a new partition 1 of type 'Linux' and of size 2 GiB.

>>> Command (m for help): n
>>> Select (default p): p
>>> Partition number (2-4, default 2): 2
>>> First sector (4196352-5242879, default 4196352):
>>> Last sector, +/-sectors or +/-size{K,M,G,T,P} (4196352-5242879, default 5242879):
>>> Created a new partition 2 of type 'Linux' and of size 511 MiB.

>>> Command (m for help): w
>>> The partition table has been altered.

lsblk | grep sdb
>>> sdb                         8:16   0  2.5G  0 disk
>>> ├─sdb1                      8:17   0    2G  0 part
>>> └─sdb2                      8:18   0  511M  0 part
```

#### 5. Используя `sfdisk`, перенесите данную таблицу разделов на второй диск.

---
```bash
sudo sfdisk -d /dev/sdb | sudo sfdisk /dev/sdc
lsblk | grep 'sd[bc]'

>>> sdb                         8:16   0  2.5G  0 disk
>>> ├─sdb1                      8:17   0    2G  0 part
>>> └─sdb2                      8:18   0  511M  0 part
>>> sdc                         8:32   0  2.5G  0 disk
>>> ├─sdc1                      8:33   0    2G  0 part
>>> └─sdc2                      8:34   0  511M  0 part
```

#### 6. Соберите `mdadm` RAID1 на паре разделов 2 Гб.

---
```bash
sudo mdadm --create --verbose /dev/md0 -l 1 -n 2 /dev/sd{b1,c1}
lsblk | grep -E '(sd[bc]|md)'

>>> sdb                         8:16   0  2.5G  0 disk
>>> ├─sdb1                      8:17   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> └─sdb2                      8:18   0  511M  0 part
>>> sdc                         8:32   0  2.5G  0 disk
>>> ├─sdc1                      8:33   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> └─sdc2                      8:34   0  511M  0 part
```

#### 7. Соберите `mdadm` RAID0 на второй паре маленьких разделов.

---
```bash
sudo mdadm --create --verbose /dev/md1 -l 0 -n 2 /dev/sd{b2,c2}
lsblk | grep -E '(sd[bc]|md)'

>>> sdb                         8:16   0  2.5G  0 disk
>>> ├─sdb1                      8:17   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> └─sdb2                      8:18   0  511M  0 part
>>>   └─md1                     9:1    0 1018M  0 raid0
>>> sdc                         8:32   0  2.5G  0 disk
>>> ├─sdc1                      8:33   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> └─sdc2                      8:34   0  511M  0 part
>>>   └─md1                     9:1    0 1018M  0 raid0
```

#### 8. Создайте 2 независимых PV на получившихся md-устройствах.

---
```bash
sudo pvcreate /dev/md0 /dev/md1

>>>  Physical volume "/dev/md0" successfully created.
>>>  Physical volume "/dev/md1" successfully created.

sudo pvdisplay

...
>>>    --- Physical volume ---
>>>    PV Name               /dev/md0
>>>    VG Name               vg01
>>>    PV Size               <2.00 GiB / not usable 0
>>>    Allocatable           yes
>>>    PE Size               4.00 MiB
>>>    Total PE              511
>>>    Free PE               511
>>>    Allocated PE          0
>>>    PV UUID               y4ypgs-e8fj-Zky2-ZnU6-ru1m-v25A-q6JN03

>>>    --- Physical volume ---
>>>    PV Name               /dev/md1
>>>    VG Name               vg01
>>>    PV Size               1018.00 MiB / not usable 2.00 MiB
>>>    Allocatable           yes
>>>    PE Size               4.00 MiB
>>>    Total PE              254
>>>    Free PE               254
>>>    Allocated PE          0
>>>    PV UUID               bdeL4X-I7Fl-WZC6-hTT7-kI24-Zc0p-UXYD95
```

#### 9. Создайте общую volume-group на этих двух PV.

---
```bash
sudo vgcreate vg01 /dev/md0 /dev/md1

>>>  Volume group "vg01" successfully created

sudo vgdisplay

...
>>>  --- Volume group ---
>>>  VG Name               vg01
>>>  System ID
>>>  Format                lvm2
>>>  Metadata Areas        2
>>>  Metadata Sequence No  1
>>>  VG Access             read/write
>>>  VG Status             resizable
>>>  MAX LV                0
>>>  Cur LV                0
>>>  Open LV               0
>>>  Max PV                0
>>>  Cur PV                2
>>>  Act PV                2
>>>  VG Size               <2.99 GiB
>>>  PE Size               4.00 MiB
>>>  Total PE              765
>>>  Alloc PE / Size       0 / 0
>>>  Free  PE / Size       765 / <2.99 GiB
>>>  VG UUID               Ga9zek-Jksk-YuUt-TQIe-Qdmt-pxpu-eeMzTF
```

#### 10. Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.

---
```bash
sudo lvcreate -L100 -n lv01 vg01 /dev/md1

>>>  Logical volume "lv01" created.

sudo lvdisplay

...
>>>  --- Logical volume ---
>>>  LV Path                /dev/vg01/lv01
>>>  LV Name                lv01
>>>  VG Name                vg01
>>>  LV UUID                ffFKfn-BPmf-eaDE-IaM5-YevH-TL1l-apwBq9
>>>  LV Write Access        read/write
>>>  LV Creation host, time vagrant, 2022-01-27 05:04:42 +0000
>>>  LV Status              available
>>>  # open                 0
>>>  LV Size                100.00 MiB
>>>  Current LE             25
>>>  Segments               1
>>>  Allocation             inherit
>>>  Read ahead sectors     auto
>>>  - currently set to     4096
>>>  Block device           253:1
```

#### 11. Создайте `mkfs.ext4` ФС на получившемся LV.

---
```bash
sudo mkfs.ext4 /dev/vg01/lv01

>>>  mke2fs 1.45.5 (07-Jan-2020)
>>>  Creating filesystem with 25600 4k blocks and 25600 inodes

>>>  Allocating group tables: done
>>>  Writing inode tables: done
>>>  Creating journal (1024 blocks): done
>>>  Writing superblocks and filesystem accounting information: done

```

#### 12. Смонтируйте этот раздел в любую директорию, например, `/tmp/new`.

---
```bash
mkdir /tmp/new
sudo mount /dev/vg01/lv01 /tmp/new
```

#### 13. Поместите туда тестовый файл, например `wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz`.

---
```bash
sudo wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz

>>> ‘/tmp/new/test.gz’ saved [22106552/22106552]
```

#### 14. Прикрепите вывод `lsblk`.

---

```bash
lsblk

NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
loop0                       7:0    0 55.4M  1 loop  /snap/core18/2128
loop1                       7:1    0 70.3M  1 loop  /snap/lxd/21029
loop2                       7:2    0 32.3M  1 loop  /snap/snapd/12704
loop3                       7:3    0 55.5M  1 loop  /snap/core18/2284
loop4                       7:4    0 43.4M  1 loop  /snap/snapd/14549
loop5                       7:5    0 61.9M  1 loop  /snap/core20/1270
loop6                       7:6    0 67.2M  1 loop  /snap/lxd/21835
sda                         8:0    0   64G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part  /boot
└─sda3                      8:3    0   63G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0 31.5G  0 lvm   /
sdb                         8:16   0  2.5G  0 disk
├─sdb1                      8:17   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
└─sdb2                      8:18   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
    └─vg01-lv01           253:1    0  100M  0 lvm   /tmp/new
sdc                         8:32   0  2.5G  0 disk
├─sdc1                      8:33   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
└─sdc2                      8:34   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
    └─vg01-lv01           253:1    0  100M  0 lvm   /tmp/new
```

#### 15. Протестируйте целостность файла:

```bash
root@vagrant:~# gzip -t /tmp/new/test.gz
root@vagrant:~# echo $?
0
```
---

```bash
gzip -t /tmp/new/test.gz
echo $?

>>> 0
```

#### 16. Используя pvmove, переместите содержимое PV с RAID0 на RAID1.

---

```bash
sudo pvmove -n lv01 /dev/md1 /dev/md0
lsblk

...
>>> sdb                         8:16   0  2.5G  0 disk
>>> ├─sdb1                      8:17   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> │   └─vg01-lv01           253:1    0  100M  0 lvm   /tmp/new
>>> └─sdb2                      8:18   0  511M  0 part
>>>   └─md1                     9:1    0 1018M  0 raid0
>>> sdc                         8:32   0  2.5G  0 disk
>>> ├─sdc1                      8:33   0    2G  0 part
>>> │ └─md0                     9:0    0    2G  0 raid1
>>> │   └─vg01-lv01           253:1    0  100M  0 lvm   /tmp/new
>>> └─sdc2                      8:34   0  511M  0 part
>>>   └─md1                     9:1    0 1018M  0 raid0
```

#### 17. Сделайте `--fail` на устройство в вашем RAID1 md.

---

```bash
sudo mdadm /dev/md0 --fail /dev/sdb1

>>> mdadm: set /dev/sdb1 faulty in /dev/md0

sudo mdadm --detail /dev/md0

...
>>>             State : clean, degraded
...
>>>    Number   Major   Minor   RaidDevice State
>>>       -       0        0        0      removed
>>>       1       8       33        1      active sync   /dev/sdc1
>>>
>>>       0       8       17        -      faulty   /dev/sdb1
```

#### 18. Подтвердите выводом `dmesg`, что RAID1 работает в деградированном состоянии.

---

```bash
dmesg

>>> [Wed Jan 27 06:05:33 2022] md/raid1:md0: Disk failure on sdb1, disabling device.
>>>                            md/raid1:md0: Operation continuing on 1 devices.
```

#### 19. Протестируйте целостность файла, несмотря на "сбойный" диск он должен продолжать быть доступен:

```bash
root@vagrant:~# gzip -t /tmp/new/test.gz
root@vagrant:~# echo $?
0
```
---

```bash
gzip -t /tmp/new/test.gz
echo $?

>>> 0
```

#### 20. Погасите тестовый хост, `vagrant destroy`.

---
```bash
vagrant destroy
```