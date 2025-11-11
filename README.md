# Управление программными RAID-массивами в Linux

### Название задания
Работа с mdadm

### Текст задания
1. Добавить в виртуальную машину несколько дисков.
2. Собрать RAID-0/1/5/10 на выбор.
3. Сломать и починить RAID.
4. Создать GPT-таблицу и пять разделов. Смонтировать разделы в системе.


В инструкции рассмотрены:
1. Создание массива RAID-5.
2. Восстановление RAID-массива после тестового сбоя.
3. Создание GPT-таблицы и монтирование разделов в системе.


Все дальнейшие действия были проверены в VirtualBox 7.1.4, на которой развёрнут Ubuntu Server 24.04.3 и клиентской машине с Ubuntu 24.04.1 Desktop. В настройках виртуальной машины добавлены 5 дисков по 1GB.

### Создание массива RAID-5

Пример создания массива RAID-5 из четырёх дисков:

    /dev/sdb
    /dev/sdc
    /dev/sdd
    /dev/sde


Чтобы создать RAID-массив:

1. В терминале выполните команду для определения блочных устройств:
```
servjul@ubuntuserver2:~$ lsblk
```
Результат вывода:

```bash
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 11,5G  0 lvm  /
sdb                         8:16   0    1G  0 disk 
sdc                         8:32   0    1G  0 disk 
sdd                         8:48   0    1G  0 disk 
sde                         8:64   0    1G  0 disk 
sdf                         8:80   0    1G  0 disk 
sr0                        11:0    1 1024M  0 rom 
```

2. Перед созданием RAID-массива удалите служебную информацию, определяющую структуру и файловую систему:
```
servjul@ubuntuserver2:~$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e}
```
Результат вывода:

```bash
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
```

3. Cоздайте массив RAID-5 c помощью ключа --create:

```
sudo mdadm --create --verbose /dev/md0 -l 5 -n 4 /dev/sd{b,c,d,e}
```

где:

    /dev/md0 — новый массив;
    -l 5 — уровень RAID-массива;
    -n 4 — количество дисков, из которых собирается массив;
    /dev/sd{b,c,d,e} — диски.


Результат вывода:

```bash
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 1046528K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

```

4. Проверьте инициализацию RAID-массива с помощью команды:

```
servjul@ubuntuserver2:~$ cat /proc/mdstat

```
Результат вывода:

```bash
Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear] 
md0 : active raid5 sde[4] sdd[2] sdc[1] sdb[0]
      3139584 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      
unused devices: <none>
```

Для получения подробной информации о созданном RAID-массиве выполните команду:
```
servjul@ubuntuserver2:~$ sudo mdadm -D /dev/md0

```
Результат вывода:

```bash
/dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 11 17:36:14 2025
        Raid Level : raid5
        Array Size : 3139584 (2.99 GiB 3.21 GB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Tue Nov 11 17:36:28 2025
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntuserver2:0  (local to host ubuntuserver2)
              UUID : 0814b66f:35134fdb:c6f9ecf8:d882a934
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde
```

### Восстановление RAID-массива после тестового сбоя.

1. Для создания тестового сбоя в RAID-массиве выведете из строя одно из блочных устройств:

```
servjul@ubuntuserver2:~$ sudo  mdadm /dev/md0 --fail /dev/sde

```
Результат вывода:

```bash
mdadm: set /dev/sde faulty in /dev/md0
```
2. Проверьте состояние RAID-массива с помощью команд `cat /proc/mdstat` и `mdadm -D /dev/md0`:

```
servjul@ubuntuserver2:~$ cat /proc/mdstat
```
Результат вывода:

```bash
Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear] 
md0 : active raid5 sde[4](F) sdd[2] sdc[1] sdb[0]
      3139584 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/3] [UUU_]
      
unused devices: <none>

```

```
servjul@ubuntuserver2:~$ sudo mdadm -D /dev/md0
```
Результат вывода:

```bash
/dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 11 17:36:14 2025
        Raid Level : raid5
        Array Size : 3139584 (2.99 GiB 3.21 GB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Tue Nov 11 17:55:03 2025
             State : clean, degraded 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntuserver2:0  (local to host ubuntuserver2)
              UUID : 0814b66f:35134fdb:c6f9ecf8:d882a934
            Events : 20

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed

       4       8       64        -      faulty   /dev/sde

```

3. Удалите диск со статусом faulty:

```
servjul@ubuntuserver2:~$ sudo mdadm /dev/md0 --remove /dev/sde
```
Результат вывода:

```bash
mdadm: hot removed /dev/sde from /dev/md0
```

4. Добавьте новый диск в RAID (представим, что мы вставили новый диск в сервер):

```
servjul@ubuntuserver2:~$ sudo mdadm /dev/md0 --add /dev/sde
```
Результат вывода:

```bash
mdadm: added /dev/sde
```

5. Проверьте состояние RAID-массива с помощью команды:

```
servjul@ubuntuserver2:~$ sudo mdadm -D /dev/md0
```
Результат вывода:

```bash
/dev/md0:
           Version : 1.2
     Creation Time : Tue Nov 11 17:36:14 2025
        Raid Level : raid5
        Array Size : 3139584 (2.99 GiB 3.21 GB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Tue Nov 11 18:05:02 2025
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : ubuntuserver2:0  (local to host ubuntuserver2)
              UUID : 0814b66f:35134fdb:c6f9ecf8:d882a934
            Events : 40

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       4       8       64        3      active sync   /dev/sde
```

### Создание GPT-таблицы и монтирование разделов в системе

1. Создайте раздел GPT на RAID с помощью команды:

```
servjul@ubuntuserver2:~$ sudo parted -s /dev/md0 mklabel gpt
```

2. Создайте разделы:

```
sudo parted /dev/md0 mkpart primary ext4 0% 20%
sudo parted /dev/md0 mkpart primary ext4 20% 40%
sudo parted /dev/md0 mkpart primary ext4 40% 60%
sudo parted /dev/md0 mkpart primary ext4 60% 80%
sudo parted /dev/md0 mkpart primary ext4 80% 100%
```

3. Создайте файловую систему Linux на созданных выше разделах: 

```
servjul@ubuntuserver2:~$ for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
```
Результат вывода:

```bash
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 156672 4k blocks and 39200 inodes
Filesystem UUID: 1fe397ef-7b8f-4308-ac10-1fc21d75d65a
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 157056 4k blocks and 39280 inodes
Filesystem UUID: b83c2ee9-5cca-4b45-be15-f07d49dacc5b
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 156672 4k blocks and 39200 inodes
Filesystem UUID: 8de8683a-7351-4575-85d7-e539cb82111d
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 157056 4k blocks and 39280 inodes
Filesystem UUID: 9ca17d59-e2cb-4c2c-a953-508107150e6b
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 156672 4k blocks and 39200 inodes
Filesystem UUID: a016bfa4-2cf2-4251-b4dc-514a94b072ea
Superblock backups stored on blocks: 
	32768, 98304

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

```

4. Смонтируйте разделы по каталогам:

```
servjul@ubuntuserver2:~$ sudo mkdir -p /raid/part{1,2,3,4,5}
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

```

5. Проверьте состояние о блочных устройствах в системе с помощью команды:

```
servjul@ubuntuserver2:~$ lsblk
```
Результат вывода:

```bash
NAME                      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                         8:0    0    25G  0 disk  
├─sda1                      8:1    0     1M  0 part  
├─sda2                      8:2    0     2G  0 part  /boot
└─sda3                      8:3    0    23G  0 part  
  └─ubuntu--vg-ubuntu--lv 252:0    0  11,5G  0 lvm   /
sdb                         8:16   0     1G  0 disk  
└─md0                       9:0    0     3G  0 raid5 
  ├─md0p1                 259:1    0   612M  0 part  
  ├─md0p2                 259:2    0 613,5M  0 part  
  ├─md0p3                 259:3    0   612M  0 part  
  ├─md0p4                 259:8    0 613,5M  0 part  
  └─md0p5                 259:9    0   612M  0 part  
sdc                         8:32   0     1G  0 disk  
└─md0                       9:0    0     3G  0 raid5 
  ├─md0p1                 259:1    0   612M  0 part  
  ├─md0p2                 259:2    0 613,5M  0 part  
  ├─md0p3                 259:3    0   612M  0 part  
  ├─md0p4                 259:8    0 613,5M  0 part  
  └─md0p5                 259:9    0   612M  0 part  
sdd                         8:48   0     1G  0 disk  
└─md0                       9:0    0     3G  0 raid5 
  ├─md0p1                 259:1    0   612M  0 part  
  ├─md0p2                 259:2    0 613,5M  0 part  
  ├─md0p3                 259:3    0   612M  0 part  
  ├─md0p4                 259:8    0 613,5M  0 part  
  └─md0p5                 259:9    0   612M  0 part  
sde                         8:64   0     1G  0 disk  
└─md0                       9:0    0     3G  0 raid5 
  ├─md0p1                 259:1    0   612M  0 part  
  ├─md0p2                 259:2    0 613,5M  0 part  
  ├─md0p3                 259:3    0   612M  0 part  
  ├─md0p4                 259:8    0 613,5M  0 part  
  └─md0p5                 259:9    0   612M  0 part  
sdf                         8:80   0     1G  0 disk  
sr0                        11:0    1  1024M  0 rom   

```

Работа с mdadm завершена.