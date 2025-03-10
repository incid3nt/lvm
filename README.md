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
Как видим, оба диска отображаются. Разберем информацию из вывода команды:

PV Name – имя диска или раздела
VG Name – группа томов, в которую данный диск входит (мы пока группу не создали)
PV Size – размер диска или размера
Allocatable – распределение по группам. В нашем случае распределения не было, поэтому указано NO
PE Size – размер физического фрагмента. Если диск не добавлен ни в одну группу, значение всегда будет 0
Total PE – количество физических фрагментов
Free PE — количество свободных физических фрагментов
Allocated PE – распределенные фрагменты
PV UUID – идентификатор раздела

посмотрим группы томов: vgdisplay:
```
root@debian:/home/oleg# /usr/sbin/vgdisplay
  --- Volume group ---
  VG Name               debian-vg
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  6
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                5
  Open LV               5
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <19,52 GiB
  PE Size               4,00 MiB
  Total PE              4997
  Alloc PE / Size       4997 / <19,52 GiB
  Free  PE / Size       0 / 0
  VG UUID               kJJoKh-m9rj-wpFM-Oa4P-0mOD-jUC5-bO1FHh

```
расширим vg за счет нового диска sdb
```
root@debian:/home/oleg# /usr/sbin/vgextend debian-vg /dev/sdb
  Volume group "debian-vg" successfully extended
```
проверим
```

root@debian:/home/oleg# /usr/sbin/vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  debian-vg   2   5   0 wz--n- <39,52g <20,00g

```
как видим VG Size увеличился

```
root@debian:/home/oleg# df -h
Файловая система            Размер Использовано  Дост Использовано% Cмонтировано в
udev                          960M            0  960M            0% /dev
tmpfs                         197M         600K  197M            1% /run
/dev/mapper/debian--vg-root   3,9G         1,4G  2,4G           37% /
tmpfs                         984M            0  984M            0% /dev/shm
tmpfs                         5,0M            0  5,0M            0% /run/lock
/dev/mapper/debian--vg-tmp    332M          10K  309M            1% /tmp
/dev/mapper/debian--vg-home    13G          44K   12G            1% /home
/dev/mapper/debian--vg-var    1,6G         333M  1,2G           22% /var
/dev/sda1                     455M          99M  332M           23% /boot
tmpfs                         197M            0  197M            0% /run/user/1000
```
расширим tmp на 2G
```
root@debian:/home/oleg# /usr/sbin/lvextend -L+2G /dev/mapper/debian--vg-tmp
  Size of logical volume debian-vg/tmp changed from 364,00 MiB (91 extents) to <2,36 GiB (603 extents).
  Logical volume debian-vg/tmp successfully resized.
```
расширим:
```
resize2fs /dev/mapper/debian--vg-tmp
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/debian--vg-tmp is mounted on /tmp; on-line resizing required
old_desc_blocks = 3, new_desc_blocks = 19
The filesystem on /dev/mapper/debian--vg-tmp is now 2469888 (1k) blocks long.
```
```
root@debian:/home/oleg# df -h | grep /dev/mapper/debian--vg-tmp
/dev/mapper/debian--vg-tmp    2,2G          10K  2,1G            1% /tmp
```
как мы видим все прошло успешно.

создать том с именем backup на 5G в группе debian-vg
```
/usr/sbin/lvcreate -L 5G -n backup debian-vg
  Logical volume "backup" created.
```
root@debian:/# /usr/sbin/lvscan
  ACTIVE            '/dev/debian-vg/root' [4,03 GiB] inherit
  ACTIVE            '/dev/debian-vg/var' [6,65 GiB] inherit
  ACTIVE            '/dev/debian-vg/swap_1' [976,00 MiB] inherit
  ACTIVE            '/dev/debian-vg/tmp' [<2,36 GiB] inherit
  ACTIVE            '/dev/debian-vg/home' [<12,53 GiB] inherit
  ACTIVE            '/dev/debian-vg/backup' [5,00 GiB] inherit
