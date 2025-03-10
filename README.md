```
LVM (Logical Volume Manager) – подсистема операционных систем Linux, позволяющая использовать разные области физического жесткого диска или разных жестких дисков как один логический том. LVM встроена в ядро Linux и реализуется на базе device mapper.


Главные преимущества LVM – высокий уровень абстракции от физических дисков, гибкость и масштабируемость. Вы можете на лету изменять размер логического тома, добавлять (и удалять) новые диски. Для LVM томов поддерживается зекалирование, снапшоты (persistent snapshot) и striping (расслоение данных между несколькими дисками с целью увеличения производительности).

В данной статье мы рассмотрим использование LVM разделов на примере Linux CentOS 8, покажем процесс объединения двух дисков в одну группу LVM, посмотрим как создавать группы, тома, монтировать, расширять и уменьшать размер LVM разделов.
```
Прежде всего нужно разобраться с уровнями дисковых абстракций LVM.

Physical Volume (PV) – физический уровень. Физические диски инициализируются для использования в LVM.
Volume Group (VG) – уровень группы томов. Инициализированные диски объединяются в логические группы с именем.
Logical Volume (LV) — создается логический том на группе томов, на котором размещается файловая система и данные.

![lvm](https://github.com/incid3nt/lvm/blob/main/screen/arhitektura-i-urovni-abstracii-lvm-v-linux.png)

Итак, у нас имеется vm с двумя дисками:
```
root@debian:/home/oleg# /usr/sbin/fdisk -l
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x708ec17f

Device     Boot   Start      End  Sectors  Size Id Type
/dev/sda1  *       2048   999423   997376  487M 83 Linux
/dev/sda2       1001470 41940991 40939522 19,5G  5 Extended
/dev/sda5       1001472 41940991 40939520 19,5G 8e Linux LVM


Disk /dev/mapper/debian--vg-root: 4,03 GiB, 4328521728 bytes, 8454144 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/debian--vg-var: 1,65 GiB, 1774190592 bytes, 3465216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/debian--vg-swap_1: 976 MiB, 1023410176 bytes, 1998848 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/debian--vg-tmp: 364 MiB, 381681664 bytes, 745472 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/debian--vg-home: 12,53 GiB, 13451132928 bytes, 26271744 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```
Как вы видите, у меня доступны два диска /dev/sda и /dev/sdb .

sda уже задействован в lvm

При настройке LVM на своем виртуальном или физическом сервере, используйте свою маркировку дисков.

Чтобы диски были доступны для LVM, их нужно пометить (инициализировать) утилитой pvcreate:

pvcreate /dev/sdb

```
root@debian:/home/oleg# /usr/sbin/pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```
Теперь, чтобы убедиться, что данные диски можно использовать для LVM, введите команду pvdisplay:
```
root@debian:/home/oleg# /usr/sbin/pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda5
  VG Name               debian-vg
  PV Size               19,52 GiB / not usable 2,00 MiB
  Allocatable           yes (but full)
  PE Size               4,00 MiB
  Total PE              4997
  Free PE               0
  Allocated PE          4997
  PV UUID               bb8jqs-5ITR-Jnlx-QXI0-Rpj1-r938-esKs0q

  "/dev/sdb" is a new physical volume of "20,00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb
  VG Name
  PV Size               20,00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               bjbWkG-ndOr-Aisi-ZQdt-qCqg-YrXJ-eICEt2
```