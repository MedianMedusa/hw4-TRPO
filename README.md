# ДЗ 4 по технологиям разработки ПО
## Задание
Надо было по мануалу попрактиковаться с массивами RAID: собрать какой-нибудь, сломать и починить его, заставить работать при перезагрузке. ТЗ:
<details>
  <summary>Раскрыть</summary>  


Практика с mdadm:
- Cобрать R0/R5/R10 на выбор
- Сломать и починить RAID;
- Проверить, что рейд собирается при перезагрузке;
- Cоздать на созданном RAID-устройстве раздел и файловую систему;
- Добавить запись в fstab для автомонтирования при перезагрузке.

Ставим нужные пакеты:
yum install mdadm

Создаём массив (RAID10 для примера):
mdadm --create /dev/md0 -l 10 -n 4 /dev/loop{0..3}

Проверяем состояние:
cat /proc/mdstat
mdadm --detail /dev/md0

Пометим диск как сбойный и извлечём его мз массива:
mdadm /dev/md0 --fail /dev/loop0 
mdadm /dev/md0 --remove /dev/loop0
cat /proc/mdstat
mdadm --detail /dev/md0

Добавим чистый диск в массив взамен удалённого:
mdadm --add /dev/md0 /dev/loop4
cat /proc/mdstat
mdadm --detail /dev/md0

Добавим ещё диск:
mdadm --add /dev/md0 /dev/loop0

Создадим конфигурационный файл:
mdadm --detail --scan > /etc/mdadm.conf

Остановим и запустим массив:
mdadm --stop /dev/md0
mdadm --assemble /dev/md0

Попробуем перезагрузиться, если практикуетесь не на loop.
На loop-устройствах массив, конечно же, не соберётся, т.к. они отсутвуют при запуске ОС: loop придётся запустить заново и собрать массив руками.

Создаём раздел (можете выбрать любой способ):
fdisk /dev/md0

Создаём файловую систему:
mkfs.ext4 /dev/md0p1

Редактируем файл fstab по образу того, что там есть (man fstab):
vi /etc/fstab
/dev/md0p1    /mnt    ext4    defaults    0   0
(лучше, конечно, узнать UUID командой blkid и успользовать UUID)

Выполнить монтирование:
mount -a 
(если предыдущие шаги корректные, то если выполнить mount вы увидите строку вида /dev/md0p1 on /mnt type ext4 (rw,relatime,seclabel,stripe=256,data=ordered))

</details>

## Инструкция
1. Настраиваем ВМ - добавляем диски. 
 - В учебных целях использую vagrant, поэтому его стоит настроить перед запуском ВМ и добавить в Vagrantfile следующее:
```
config.vm.provider "virtualbox" do |vb|
     vb.memory = "1024"
     second_disk = "RaidDisk.vmdk"
     vb.customize ['createhd', '--filename', second_disk, '--size', 1024]
     vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', second_disk]
    end

```
 - `lsblk`:
```
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  10G  0 disk
└─sda1   8:1    0  10G  0 part /
sdb      8:16   0   1G  0 disk
```
 - Разделим на разделы (залезем в следующую лабу):
 ```
 sudo -i
 fdisk /dev/sdb
 ```
   Вывод `lsblk`:
  ```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0    10G  0 disk
└─sda1   8:1    0    10G  0 part /
sdb      8:16   0     1G  0 disk
├─sdb1   8:17   0 254.8M  0 part
├─sdb2   8:18   0 255.5M  0 part
├─sdb3   8:19   0 256.8M  0 part
└─sdb4   8:20   0 255.3M  0 part
  ```
2. Создаём массив RAID:
 Я выбрал RAID5. Создание любого типа массива происходит по одному и тому же сценарию.
 - Нужный пакет: `yum install mdadm`
 - Создание: `mdadm --create /dev/md0 -l 5 -n 3 /dev/sdb{2..4}`
 Проверим:
 ```
 mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Mon Dec  7 22:16:34 2020
        Raid Level : raid5
        Array Size : 517120 (505.00 MiB 529.53 MB)
     Used Dev Size : 258560 (252.50 MiB 264.77 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Mon Dec  7 22:16:41 2020
             State : clean
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : localhost.localdomain:0  (local to host localhost.localdomain)
              UUID : 5bd0db30:94df1aa4:3c25366c:e4729df8
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       18        0      active sync   /dev/sdb2
       1       8       19        1      active sync   /dev/sdb3
       3       8       20        2      active sync   /dev/sdb4
```
3. Ломаем и чиним массив:
 - Ломаем: `mdadm /dev/md0 --fail /dev/sdb4`
 - Убираем из массива сломанный: `mdadm /dev/md0 --remove /dev/sdb4`
 - Добавляем чистый: `mdadm /dev/md0 --add /dev/sdb1`
 - Смотрим, как дела: `cat proc/mdstat`:
 ```
 md0 : active raid5 sdb1[3] sdb3[1] sdb2[0]
      517120 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]

 unused devices: <none>
 ```
 - `mdadm --detail /dev/md0`:
 ```
 ...
     Number   Major   Minor   RaidDevice State
       0       8       18        0      active sync   /dev/sdb2
       1       8       19        1      active sync   /dev/sdb3
       3       8       17        2      active sync   /dev/sdb1
 ```
4. Создаём конфиг, останавливаем и запускаем массив:
 - Конфиг: `mdadm --detail --scan > /etc/mdadm.conf && cat /etc/mdadm.conf`:
 ```
 ARRAY /dev/md0 metadata=1.2 name=localhost.localdomain:0 UUID=5bd0db30:94df1aa4:3c25366c:e4729df8
 ```
 - Остановка: `mdadm --stop /dev/md0`
 - Запуск: `mdadm --assemble /dev/md0`
 - Проверка: `lsblk`:
 ```
 NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda       8:0    0    10G  0 disk
└─sda1    8:1    0    10G  0 part  /
sdb       8:16   0     1G  0 disk
├─sdb1    8:17   0 255.8M  0 part
│ └─md0   9:0    0   505M  0 raid5
├─sdb2    8:18   0 255.5M  0 part
│ └─md0   9:0    0   505M  0 raid5
├─sdb3    8:19   0 255.3M  0 part
│ └─md0   9:0    0   505M  0 raid5
└─sdb4    8:20   0   255M  0 part
```
 > Всё корректно, можно переходить к созданию файловой системы
5. Создаём ФС:
 - Создаём раздел: `fdisk /dev/md0`, проверяем:
 ```
 lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda           8:0    0    10G  0 disk
└─sda1        8:1    0    10G  0 part  /
sdb           8:16   0     1G  0 disk
├─sdb1        8:17   0 255.8M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md
├─sdb2        8:18   0 255.5M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md
├─sdb3        8:19   0 255.3M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md
└─sdb4        8:20   0   255M  0 part
```
 - Создаём файловую систему на разделе: `mkfs.ext4 /dev/md0p1`
  Вывод:
  ```
  mke2fs 1.44.3 (10-July-2018)
Creating filesystem with 516096 1k blocks and 129024 inodes
Filesystem UUID: 648a11c1-2c88-46ce-858f-52f9ee10ac07
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
```
  Запоминаем UUID, чтобы добавить его в fstab.
 - Редактируем /etc/fstab по мануалу. Добавляем строку:
 ```
 UUID=648a11c1-2c88-46ce-858f-52f9ee10ac07 /mnt ext4 defaults 0 0
 ```
 - Монтируем: `mount -a`
  Ошибок не выдало, ровно как и вообще чего-либо. Проверим:
  ```
  lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda           8:0    0    10G  0 disk
└─sda1        8:1    0    10G  0 part  /
sdb           8:16   0     1G  0 disk
├─sdb1        8:17   0 255.8M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md    /mnt
├─sdb2        8:18   0 255.5M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md    /mnt
├─sdb3        8:19   0 255.3M  0 part
│ └─md0       9:0    0   505M  0 raid5
│   └─md0p1 259:0    0   504M  0 md    /mnt
└─sdb4        8:20   0   255M  0 part
```
   > Готово! И при перезагрузке ничего не ломается :)
   
