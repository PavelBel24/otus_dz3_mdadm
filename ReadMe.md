# Работа с mdadm

&nbsp;

Цель: научиться использовать утилиту для управления программными RAID-массивами в Linux.

## Домашнее задание

Описание:  
Для выполнения домашнего задания используйте методичку

Что нужно сделать?

- добавить в Vagrantfile еще дисков;
- собрать R0/R5/R10 на выбор;
- сломать/починить raid;
- прописать собранный рейд в конф, чтобы рейд собирался при загрузке;
- создать GPT раздел и 5 партиций.

На проверку отправьте :

- измененный Vagrantfile,
    
- скрипт для создания рейда,
    
- конф для автосборки рейда при загрузке.
    
    - Доп. задание\*  
        Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться.
        
    - Задание повышенной сложности\*\*
        
        - перенести работающую систему с одним диском на RAID 1.
        - Даунтайм на загрузку с нового диска предполагается.

На проверку отправьте  
вывод команды lsblk до и после и описание хода решения (можно воспользоваться утилитой Script).

## Выполнение

### Подготовительный этап

Стенд: [Vagrantfile](file:./vbagrantfile) взят [отсюда](https://github.com/erlong15/otus-linux)  
В исходное описание VM добавлен ещё один диск SATA, объемом 50 Мб.  
Собранный стенд состоит из 5 дисков SATA по 50МБ.

#### Проверка доступных дисков.

команда `sudo fdisk -l`

```Bash
vagrant@otuslinux ~]$ sudo fdisk -l | grep -i disk
Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Disk label type: dos
Disk identifier: 0x0009ef1a
Disk /dev/sdb: 52 MB, 52428800 bytes, 102400 sectors
Disk /dev/sdc: 52 MB, 52428800 bytes, 102400 sectors
Disk /dev/sdd: 52 MB, 52428800 bytes, 102400 sectors
Disk /dev/sde: 52 MB, 52428800 bytes, 102400 sectors
Disk /dev/sdf: 52 MB, 52428800 bytes, 102400 sectors
```

альтернативный вариант

команда `sudo lshw -short | grep disk`

```Bash
vagrant@otuslinux ~]$ sudo lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb   disk        52MB VBOX HARDDISK
/0/100/d/1          /dev/sdc   disk        52MB VBOX HARDDISK
/0/100/d/2          /dev/sdd   disk        52MB VBOX HARDDISK
/0/100/d/3          /dev/sde   disk        52MB VBOX HARDDISK
/0/100/d/0.0.0      /dev/sdf   disk        52MB VBOX HARDDISK
```

#### Обнуляем супер блок

`sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}`

```
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf
```

### Создание RAID 10

#### Создаем RAID

Команда для создания RAID 10  
`mdadm --create --verbose /dev/md0 --level=10 --raid-devices=5 /dev/sd{b,c,d,e,f}`

результат выполнения

```
sudo mdadm --create --verbose /dev/md0 --level=10 --raid-devices=5 /dev/sd{b,c,d,e,f}
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 49152K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

#### Проверка

Проверяем наличие созданного RAID массива и его состояние

```
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid10] 
md0 : active raid10 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      122880 blocks super 1.2 512K chunks 2 near-copies [5/5] [UUUUU]
      
unused devices: <none>
[vagrant@otuslinux ~]$ 

```

Детали созданного RAID массива

```
[vagrant@otuslinux ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb 11 13:04:28 2024
        Raid Level : raid10
        Array Size : 122880 (120.00 MiB 125.83 MB)
     Used Dev Size : 49152 (48.00 MiB 50.33 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Feb 11 13:04:30 2024
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6e12da5c:d063f7f1:b98f197b:c399476e
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
[vagrant@otuslinux ~]$ 
```

### Конфигурирование ОС для работы с RAID массивом

#### Просмотр конфигурации массива

```
[vagrant@otuslinux ~]$ sudo mdadm --detail --scan --verbose /dev/md0
ARRAY /dev/md0 level=raid10 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=6e12da5c:d063f7f1:b98f197b:c399476e
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf
[vagrant@otuslinux ~]$ 
```

#### Формируем конфигурационный файл mdadm.conf

```
[vagrant@otuslinux ~]$ sudo su
[root@otuslinux vagrant]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux vagrant]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf 
[root@otuslinux vagrant]# cat /etc/mdadm/mdadm.conf 
DEVICE partitions
ARRAY /dev/md0 level=raid10 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=6e12da5c:d063f7f1:b98f197b:c399476e
[root@otuslinux vagrant]# 
```

### Восстановление RAID массива

#### Имитация выхода из строя диска

```
[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sdd
mdadm: set /dev/sdd faulty in /dev/md0
[root@otuslinux vagrant]# 
```

#### Проверка состояния массива

команды :

- `cat /proc/mdstat`
- `mdadm -D /dev/md0`

Контролируем :

- статус массива  
    `md0 : active raid10 sde[3] sdf[4] sdc[1] sdb[0] sdd[2](F) 122880 blocks super 1.2 512K chunks 2 near-copies [5/4] [UU_UU]`  
    `State : clean, degraded`
    
- информация о вышедшем из строя диске  
    `Failed Devices : 1`  
    `2 8 48 - faulty /dev/sdd`  
    Подробный вывод команд
    

```
[root@otuslinux vagrant]# cat /proc/mdstat
Personalities : [raid10] 
md0 : active raid10 sde[3] sdf[4] sdc[1] sdb[0] sdd[2](F)
      122880 blocks super 1.2 512K chunks 2 near-copies [5/4] [UU_UU]
 
unused devices: <none>
[root@otuslinux vagrant]# 
[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb 11 13:04:28 2024
        Raid Level : raid10
        Array Size : 122880 (120.00 MiB 125.83 MB)
     Used Dev Size : 49152 (48.00 MiB 50.33 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Feb 11 14:11:08 2024
             State : clean, degraded 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6e12da5c:d063f7f1:b98f197b:c399476e
            Events : 19

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       -       0        0        2      removed
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf

       2       8       48        -      faulty   /dev/sdd

```

#### Извлечение "вышедшего" из строя диска

```
[root@otuslinux vagrant]# mdadm /dev/md0 --remove /dev/sdd
mdadm: hot removed /dev/sdd from /dev/md0
[root@otuslinux vagrant]# 
```

#### Имитация установки "нового" диска

```
[root@otuslinux vagrant]# mdadm /dev/md0 --add /dev/sdd
mdadm: added /dev/sdd
[root@otuslinux vagrant]#
```

#### Проверка состояния массива

```
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdd[5] sde[3] sdf[4] sdc[1] sdb[0]
      122880 blocks super 1.2 512K chunks 2 near-copies [5/5] [UUUUU]
      
unused devices: <none>
[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb 11 13:04:28 2024
        Raid Level : raid10
        Array Size : 122880 (120.00 MiB 125.83 MB)
     Used Dev Size : 49152 (48.00 MiB 50.33 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Feb 11 14:24:54 2024
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6e12da5c:d063f7f1:b98f197b:c399476e
            Events : 39

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       5       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
[root@otuslinux vagrant]# 
```

Т.к. диск малого объема процесс синхронизации данных RAID массива опущен.