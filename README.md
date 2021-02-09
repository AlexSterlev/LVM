# Выполнено ДЗ №3

## В процессе сделано:
 ### Уменьшить том под / до 8G
  - Установка пакета xfsdump (используется при копировании данных)

  ````
  yum install -y xfsdump
  ````

  - Создадим временный том:

  ````
  pvcreate /dev/sdb
  vgcreate vg_root /dev/sdb
  lvcreate -n lv_root -l +100%FREE /dev/vg_root
  ````

  - Создадим файловую систему и примонтируем

  ````
  mkfs.xfs /dev/vg_root/lv_root
  mount /dev/vg_root/lv_root /mnt
  ````

  - Скопируем данные

  ````
  xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
  ````

  - Переконфигурируем grub 

  ````
  for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
  chroot /mnt/
  grub2-mkconfig -o /boot/grub2/grub.cfg
  ````

  - Обновим initrd (+ небольшой "патч")

  ````
  cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
  ````

  Дополнительно поправим /boot/grub2/grub.cfg - заменим rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

  - Перезагружаемся 

  ````
  shutdown -r now
  ````

  - Изменяем размер 

  ````
  lvremove /dev/VolGroup00/LogVol00
  lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
  ````

  - Снова форматируем и сохраняем данные

  ````
  mkfs.xfs /dev/VolGroup00/LogVol00
  mount /dev/VolGroup00/LogVol00 /mnt
  xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
  ````

  - Переконфигурируем grub 

  ````
  for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
  chroot /mnt/
  grub2-mkconfig -o /boot/grub2/grub.cfg
  cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
  ````

  - Перезагружаемся

  ````
  shutdown -r now
  ````

 - Далее

  ````
  lvremove /dev/vg_root/lv_root
  vgremove /dev/vg_root
  pvremove /dev/sdb

  [vagrant@lvm ~]$ lsblk
  NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda                        8:0    0   40G  0 disk 
  ├─sda1                     8:1    0    1M  0 part 
  ├─sda2                     8:2    0    1G  0 part /boot
  └─sda3                     8:3    0   39G  0 part 
    ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
    └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  sdb                        8:16   0   10G  0 disk  
  sdc                        8:32   0    2G  0 disk 
  sdd                        8:48   0    1G  0 disk 
  sde                        8:64   0    1G  0 disk
  ````

 ### Выделить том под /home
  - Создаем том

  ````
  lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  ````

  - Отформатируем

  ````
  mkfs.xfs /dev/VolGroup00/LogVol_Home
  ````

  - Перемонтируем

  ````
  mount /dev/VolGroup00/LogVol_Home /mnt/
  cp -aR /home/* /mnt/ 
  rm -rf /home/*
  umount /mnt
  mount /dev/VolGroup00/LogVol_Home /home/
  ````

  - Правим fstab

  ````
  echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
  ````

  - Перезагрузка

  ````
  [vagrant@lvm ~]$ lsblk
  NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda                          8:0    0   40G  0 disk 
  ├─sda1                       8:1    0    1M  0 part 
  ├─sda2                       8:2    0    1G  0 part /boot
  └─sda3                       8:3    0   39G  0 part 
    ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
    ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
    └─VolGroup00-LogVol_Home 253:3    0    2G  0 lvm  /home
  sdb                          8:16   0   10G  0 disk 
  sdc                          8:32   0    2G  0 disk 
  sdd                          8:48   0    1G  0 disk 
  sde                          8:64   0    1G  0 disk 
  ````
  

 ### Выделить том под /var (mirror)

  - Создаем зеркало

  ````
  pvcreate /dev/sdc /dev/sdd
  vgcreate vg_var /dev/sdc /dev/sdd
  lvcreate -L 950M -m1 -n lv_var vg_var
  ````

  - Форматируем и монтируем

  ````
  mkfs.ext4 /dev/vg_var/lv_var
  mount /dev/vg_var/lv_var /mnt
  cp -aR /var/* /mnt/ 
  mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
  umount /mnt
  mount /dev/vg_var/lv_var /var
  ````

  - Правим fstab

  ````
  echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
  ````

  - Перезагрузка

  ````
  [vagrant@lvm ~]$ lsblk
  NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda                          8:0    0   40G  0 disk 
  ├─sda1                       8:1    0    1M  0 part 
  ├─sda2                       8:2    0    1G  0 part /boot
  └─sda3                       8:3    0   39G  0 part 
    ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
    └─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
    └─VolGroup00-LogVol_Home 253:3    0    2G  0 lvm  /home
  sdb                          8:16   0   10G  0 disk 
  └─vg_root-lv_root          253:2    0   10G  0 lvm  
  sdc                          8:32   0    2G  0 disk 
  ├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm  
  │ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
  └─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm  
    └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
  sdd                          8:48   0    1G  0 disk 
  ├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm  
  │ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
  └─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm  
    └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
  sde                          8:64   0    1G  0 disk
  ````

 ### Восстановление из snapshot-а

 - Сгенерируем данные

 ````
 touch /home/file{1..20}
 ````

 - Создадим снэпшот

 ````
 lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
 ````

 - Удалим несколько файлов
 
 ````
 rm -f /home/file{11..20}
 ````

- Восстановим их

 ````
 umount /home
 lvconvert --merge /dev/VolGroup00/home_snap
 mount /home
 ````

Запись процесса работы со снэпшотами в файле record.txt

