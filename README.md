### Введение в работу с LVM

#### Уменьшение корневого раздела до 8ГБ:
```
[vagrant@lvm ~]$ lsblk

NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT

sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```
#### Проверяем размер занятый файловой системой и видим что тип ФС xfs.
```
[root@lvm ~]# df -Th
Filesystem                      Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00 xfs        38G  862M   37G   3% /
devtmpfs                        devtmpfs  109M     0  109M   0% /dev
tmpfs                           tmpfs     118M     0  118M   0% /dev/shm
tmpfs                           tmpfs     118M  4.6M  114M   4% /run
tmpfs                           tmpfs     118M     0  118M   0% /sys/fs/cgroup
/dev/sda2                       xfs      1014M   63M  952M   7% /boot
tmpfs                           tmpfs      24M     0   24M   0% /run/user/0
tmpfs                           tmpfs      24M     0   24M   0% /run/user/1000
```
#### Так как xfs не умеет уменьшаться необходимо будет создать новый раздел, скопировать в него данные при помощи xfsdump, удалить корневой раздел, создать новый корневой раздел на 8ГБ, восстановить в него данные.
#### Устанавливаем xfsdump
`
yum install xfsdump
`
### Создаем временный раздел для данных /
#### Создаем PV
```
[root@lvm ~]# pvcreate /dev/sdb 
  Physical volume "/dev/sdb" successfully created.
```
#### Создаем VG
```
[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
```
#### Создаем LG занимающую все пространство VG
```
[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```
#### Создаем ФС xfs на lv_root
```
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
#### Монтируем данный раздел в /mnt
`
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
`
#### Копируем данные с / в /mnt
```
[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed Jul 26 07:39:01 2023
xfsdump: session id: 22f852f4-e089-48af-bcc0-f33bdea73666
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 865530496 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Wed Jul 26 07:39:01 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 22f852f4-e089-48af-bcc0-f33bdea73666
xfsrestore: media id: 7dba0473-72ee-4132-a271-ac39dca720ab
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2735 directories and 23682 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 842519368 bytes
xfsdump: dump size (non-dir files) : 829307136 bytes
xfsdump: dump complete: 6 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 6 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
#### Смонтируем необходимые для root каталоги
`
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done 
`
#### Сделаем подмену корня при помощи chroot
`
chroot /mnt/ 
`
#### Обновим grub
```
grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
Found CentOS Linux release 7.5.1804 (Core)  on /dev/mapper/vg_root-lv_root
done
```
#### Обновим образ initrd
```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;  s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
#### Меняем логический том с / в файле grub.cfg
##### Было
`
rd.lvm.lv=VolGroup00/LogVol00
`
##### Стало
`
rd.lvm.lv=vg_root/lv_root
`
#### Выходим из chroot и перезагружаемся. Том с корнем изменился.
```
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm  
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```
#### Теперь мы можем удалить отмонтированный корневой раздел
```
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00 
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```
#### И создать новый на 8ГБ с тем же именем
```
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```
#### Создаем на нем ФС xfs
```
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
#### Монтируем данный раздел в отмонтированный после перезагрузки /mnt
`
mount /dev/VolGroup00/LogVol00 /mnt
`
#### И копируем в него данные с временного /
```
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Wed Jul 26 09:08:08 2023
xfsdump: session id: af2d1aaa-4684-4f0e-bc51-f4f6b8c7cb75
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 864172864 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_root-lv_root
xfsrestore: session time: Wed Jul 26 09:08:08 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: ed92d8cf-95ab-4c68-827f-e09e18347ca6
xfsrestore: session id: af2d1aaa-4684-4f0e-bc51-f4f6b8c7cb75
xfsrestore: media id: 5878af7f-81c5-4cb7-88b5-199753aca82f
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2739 directories and 23689 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 841174544 bytes
xfsdump: dump size (non-dir files) : 827957328 bytes
xfsdump: dump complete: 6 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 6 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
#### Снова подменяем корень с помощью chroot
#### Смонтируем необходимые для root каталоги
`
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done 
`
#### Сделаем подмену корня при помощи chroot
`
chroot /mnt/ 
`
#### Обновим grub
```
grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```
#### Обновим образ initrd
`
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;  s/.img//g"` --force; done
`
#### Перезагружаемся
`
[root@lvm ~]# reboot
`
#### Смотрим вывод lsblk. Корень переместился в уменьшенный раздел
```
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```
#### Выходим из виртуальной машины и делаем  snapshot
```
vagrant snapshot save lvm umenshiliKoren
==> lvm: Snapshotting the machine as 'umenshiliKoren'...
==> lvm: Snapshot saved! You can restore the snapshot at any time by
==> lvm: using `vagrant snapshot restore`. You can delete it using
==> lvm: `vagrant snapshot delete`.
```
### Приступаем к следующей задаче: Выделить том под /var - сделать в зеркале
#### Создаем PV на свободных дисках равного объема
```
[root@lvm ~]# pvcreate /dev/sde /dev/sdd
  Physical volume "/dev/sde" successfully created.
  Physical volume "/dev/sdd" successfully created.
```
#### Создаем VG
```
[root@lvm ~]# vgcreate vg_var /dev/sde /dev/sdd
  Volume group "vg_var" successfully created
```
#### Создаем LV
```
[root@lvm ~]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```
#### Создаем ФС
`
mkfs.ext4 /dev/vg_var/lv_var 
`
#### Монтируем раздел в mnt
`
 mount /dev/vg_var/lv_var /mnt 
`
#### Копируем раздел var в mnt т.е. в /dev/vg_var/lv_var 
`
cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/ 
`
#### Сохраняем содержимое старого var
`
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar 
`
#### Отмонтируем /mnt и смонтируем /dev/vg_var/lv_var в /var
`
mount /dev/vg_var/lv_var /var 
`
#### Правим fstab для автоматического монтирования /var.(При помощи blkid получаем UUID блочного устройства vg_var-lv_var).
`
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
`
#### Удаляем временный LVM раздел
```
lvremove /dev/vg_root/lv_root 
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[root@lvm ~]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```
### Cделать том для снапшотов
#### Генерируем файлы
`
touch /home/file{1..20}
`
#### Снимаем снапшот
```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
```
#### Удаляем часть файлов
`
rm -f /home/file{11..20}
`
#### Размонтируем /home. Ошибка "Устройство занято".  
```
umount /home 
umount: /home: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```
#### Находим процессы которые занимают устройство, убиваем их, но это не помогает, т.к. при входе в оболочку уже возникает процесс.

#### Создаем lvm раздел 
```
pvcreate /dev/sdb
Physical volume "/dev/sdb" successfully created.
vgcreate vg_home /dev/sdb
Volume group "vg_home" successfully created
lvcreate -n lv_home -l +100%FREE /dev/vg_home
WARNING: xfs signature detected on /dev/vg_home/lv_home at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/vg_home/lv_home.
  Logical volume "lv_home" created.
```
#### Создаем на нем ФС
```
mkfs.xfs /dev/vg_home/lv_home 
meta-data=/dev/vg_home/lv_home   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

```
#### Примонтируем новый раздел
`
mount /dev/vg_home/lv_home /mnt
`

#### Копируем /home на новый раздел
```
xfsdump -J - /dev/VolGroup00/LogVol_Home | xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/home
xfsdump: dump date: Wed Jul 26 10:17:31 2023
xfsdump: session id: 3ef71921-f36f-471a-8585-83bb5fe923c0
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 46720 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsdump: dumping non-directory files
xfsdump: ending media file
xfsdump: media file size 34800 bytes
xfsdump: dump size (non-dir files) : 2720 bytes
xfsdump: dump complete: 0 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /home
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol_Home
xfsrestore: session time: Wed Jul 26 10:17:31 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: 5eb1ddd6-63f2-4261-8914-c9026884d268
xfsrestore: session id: 3ef71921-f36f-471a-8585-83bb5fe923c0
xfsrestore: media id: cd5ec0ac-eab6-4e7a-ac11-2772294fa4c9
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsrestore: 3 directories and 17 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsrestore: restore complete: 0 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
#### Находим UUID нового раздела, копируем и вставляем в файл /etc/fstab вместо UUID раздела /home. Затем перезагружаем машину.
```
blkid | grep _home
/dev/mapper/vg_home-lv_home: UUID="a792f530-8e19-41ff-966a-bfb49c2b47a7" TYPE="xfs"
```
#### Таким образом мы подменяем /home для того чтобы его размонтировать. После перезагрузки наш реальный /home раздел уже отмонтирован и можно восстанавливать его из снапшота.
```
lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
```
#### Возвращаем UUID /home на место и перезагружаемся
```
blkid | grep LogVol_Home
/dev/mapper/VolGroup00-LogVol_Home: UUID="5eb1ddd6-63f2-4261-8914-c9026884d268" TYPE="xfs" 
```
#### Проверяем данные в /home
```
ls /home/
file1   file11  file13  file15  file17  file19  file20  file4  file6  file8  vagrant
file10  file12  file14  file16  file18  file2   file3   file5  file7  file9
```
#### Данные восстановлены. 
```
lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk 
├─sda1                       8:1    0    1M  0 part 
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:5    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk 
└─vg_home-lv_home          253:2    0   10G  0 lvm  
sdc                          8:32   0    2G  0 disk 
sdd                          8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1    253:6    0    4M  0 lvm  
│ └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:7    0  952M  0 lvm  
  └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm  
│ └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm  
  └─vg_var-lv_var          253:8    0  952M  0 lvm  /var
```
#### lvm раздел со снапшотом исчез т.к. был объединен с VolGroup00-LogVol_Home 
#### Можно удалить vg_home-lv_home 
```
lvremove /dev/vg_home/lv_home 
Do you really want to remove active logical volume vg_home/lv_home? [y/n]: y
  Logical volume "lv_home" successfully removed
[root@lvm ~]# vgremove /dev/vg_home
  Volume group "vg_home" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```



