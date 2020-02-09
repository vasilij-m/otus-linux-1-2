Задание 1. Добавить в Vagrantfile еще дисков. Сломать/починить raid. Собрать R0/R5/R10 на выбор.
Прописать собранный рейд в конф, чтобы рейд собирался при загрузке. Создать GPT раздел и 5 партиций.

Выведем список дисков, подключенных к ВМ:
```
[vagrant@otuslinux-1-2 ~]$ sudo lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb   disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc   disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd   disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde   disk        262MB VBOX HARDDISK
/0/100/d/4          /dev/sdf   disk        262MB VBOX HARDDISK
/0/100/d/5          /dev/sdg   disk        262MB VBOX HARDDISK
```
Перед сборкой рейда занулим суперблоки:
```
[vagrant@otuslinux-1-2 ~]$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e}
```
Создадим RAID 10:
```
[vagrant@otuslinux-1-2 ~]$ sudo mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Проверим, что рейд собрался нормально:
```
[vagrant@otuslinux-1-2 ~]$ cat /proc/mdstat
Personalities : [raid10] 
md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
[vagrant@otuslinux-1-2 ~]$ sudo mdadm -D /dev/md0 
/dev/md0:
           Version : 1.2
     Creation Time : Sun Feb  9 18:00:35 2020
        Raid Level : raid10
...
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

```
Система сама не запоминает какие рейд-массивы ей нужно создать, и какие блочные устройства в них
входят. Эта информация находится в файле `mdadm.conf`. Строки, которые в него стоит добавить, можно
узнать следующим образом:
```
[vagrant@otuslinux-1-2 ~]$ sudo mdadm --detail --scan --verbose
ARRAY /dev/md0 level=raid10 num-devices=4 metadata=1.2 name=otuslinux-1-2:0 UUID=e829d150:ad096e94:44a909af:cbb88c83
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde
```
Создадим файл `mdadm.conf` и помести в него нужную информацию:
```
[vagrant@otuslinux-1-2 ~]$ echo "DEVICE partitions" | sudo tee -a /etc/mdadm.conf
[vagrant@otuslinux-1-2 ~]$ sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' | sudo tee -a /etc/mdadm.conf
```
Сломаем рейд, пометив один из дисков как неисправный:
```
[vagrant@otuslinux-1-2 ~]$ sudo mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0
```
В результате на одну букву U в выводе информации о рейде станет меньше:
```
[vagrant@otuslinux-1-2 ~]$ cat /proc/mdstat
Personalities : [raid10] 
md0 : active raid10 sdc[1](F) sde[3] sdd[2] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/3] [U_UU]

[vagrant@otuslinux-1-2 ~]$ sudo mdadm -D /dev/md0
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       -       0        0        1      removed
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       1       8       32        -      faulty   /dev/sdc

```
Заменим отказавший диск в рейд другим диском:
```
[vagrant@otuslinux-1-2 ~]$ sudo mdadm /dev/md0 --remove /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0

[vagrant@otuslinux-1-2 ~]$ sudo mdadm /dev/md0 --add /dev/sdf
mdadm: added /dev/sdf
```
После этого произойдет процесс ребилда рейда:
```
[vagrant@otuslinux-1-2 ~]$ cat /proc/mdstat
Personalities : [raid10] 
md0 : active raid10 sdf[4] sde[3] sdd[2] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
```
Создадим на нашем рейде GPT раздел:
```
[vagrant@otuslinux-1-2 ~]$ sudo parted -s /dev/md0 mklabel gpt
```
Создадим партиции:
```
[vagrant@otuslinux-1-2 ~]$ sudo parted /dev/md0 mkpart primary ext4 0% 20%
[vagrant@otuslinux-1-2 ~]$ sudo parted /dev/md0 mkpart primary ext4 20% 40%
[vagrant@otuslinux-1-2 ~]$ sudo parted /dev/md0 mkpart primary ext4 40% 60%
[vagrant@otuslinux-1-2 ~]$ sudo parted /dev/md0 mkpart primary ext4 60% 80%
[vagrant@otuslinux-1-2 ~]$ sudo parted /dev/md0 mkpart primary ext4 80% 100%
```
Далее создаем на этих партициях файловую систему:
```
[vagrant@otuslinux-1-2 ~]$ for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
```
Смонтируем полученные файловые системы по каталогам:
```
[vagrant@otuslinux-1-2 ~]$ sudo mkdir -p /raid/part{1,2,3,4,5}

[vagrant@otuslinux-1-2 ~]$ for i in $(seq 1 5); do sudo mount /dev/md0p$i /raid/part$i; done
```



