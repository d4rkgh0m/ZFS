# ЗАДАНИЕ: Практические навыки работы с ZFS
### Определить алгоритм с наилучшим сжатием:
### Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
### создать 4 файловых системы на каждой применить свой алгоритм сжатия;
### для сжатия использовать либо текстовый файл, либо группу файлов.
### Определить настройки пула.
### С помощью команды zfs import собрать pool ZFS.
### Командами zfs определить настройки:  
### - размер хранилища;    
### - тип pool;    
### - значение recordsize;   
### - какое сжатие используется;   
### - какая контрольная сумма используется.
### Работа со снапшотами:
### скопировать файл из удаленной директории;
### восстановить файл локально. zfs receive;
### найти зашифрованное сообщение в файле secret_message.
### Создаем виртуальную машину
### Создаем Vagrantfile по предложенному шаблону
### Запускаем 
> vagrant up
### после развертывания виртуальной машины и установки пакетов указанных в provision
### заходим на виртуалку
> vagrant ssh
### Переходим в root пользователя: 
> sudo -i 
##  1. Определение алгоритма с наилучшим сжатием
### Смотрим список всех дисков, которые есть в виртуальной машине: 
> lsblk
### Вывод команды:
```ruby
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT 
sda      8:0    0   40G  0 disk  
└─sda1   8:1    0   40G  0 part / 
sdb      8:16   0  512M  0 disk   
sdc      8:32   0  512M  0 disk   
sdd      8:48   0  512M  0 disk   
sde      8:64   0  512M  0 disk   
sdf      8:80   0  512M  0 disk   
sdg      8:96   0  512M  0 disk   
sdh      8:112  0  512M  0 disk   
sdi      8:128  0  512M  0 disk
```  
### Создаём пул из двух дисков в режиме RAID 1:
> zpool create otus1 mirror /dev/sdb /dev/sdc
### Создадим ещё 3 пула:
> zpool create otus2 mirror /dev/sdd /dev/sde \
> zpool create otus3 mirror /dev/sdf /dev/sdg \
> zpool create otus4 mirror /dev/sdh /dev/sdi 
### Смотрим информацию о пулах:
> zpool list
### Вывод команды:
```ruby
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT 
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  - 
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  - 
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  - 
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
### Добавим разные алгоритмы сжатия в каждую файловую систему:
### Алгоритм lzjb:
> zfs set compression=lzjb otus1
### Алгоритм lz4:
> zfs set compression=lz4 otus2
### Алгоритм gzip: 
> zfs set compression=gzip-9 otus3
### Алгоритм zle:  
> zfs set compression=zle otus4
### Проверим, что все файловые системы имеют разные методы сжатия:
> zfs get all | grep compression
### Вывод команды:
```ruby
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
### Скачаем один и тот же текстовый файл во все пулы:
> for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
### Проверим, что файл был скачан во все пулы:
> ls -l /otus*
### Вывод команды:
```ruby
/otus1:
total 22063
-rw-r--r--. 1 root root 40997929 Dec  2 09:17 pg2600.converter.log 
/otus2:
total 17992
-rw-r--r--. 1 root root 40997929 Dec  2 09:17 pg2600.converter.log 
/otus3:
total 10959
-rw-r--r--. 1 root root 40997929 Dec  2 09:17 pg2600.converter.log 
/otus4:
total 40066
-rw-r--r--. 1 root root 40997929 Dec  2 09:17 pg2600.converter.log
```
### Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
> zfs list
### Вывод команды:
```ruby
NAME    USED  AVAIL     REFER  MOUNTPOINT 
otus1  21.6M   330M     21.6M  /otus1 
otus2  17.7M   334M     17.6M  /otus2 
otus3  10.8M   341M     10.7M  /otus3 
otus4  39.2M   313M     39.2M  /otus4
```
> zfs get all | grep compressratio | grep -v ref
### Вывод команды:
```ruby
otus1  compressratio   1.81x   - 
otus2  compressratio   2.22x   - 
otus3  compressratio   3.65x   - 
otus4  compressratio   1.00x   -
```
### Итак алгоритм gzip-9 самый эффективный по сжатию.
## 2. Определение настроек пула
### Скачиваем архив в домашний каталог:
> wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
### Разархивируем его:
> tar -xzvf archive.tar.gz
### Вывод команды:
```ruby
zpoolexport/ 
zpoolexport/filea 
zpoolexport/fileb
```
### Проверим, возможно ли импортировать данный каталог в пул:
> zpool import -d zpoolexport/
### Вывод команды:
```ruby
pool: otus 
  id: 6554193320433390805
    state: ONLINE 
 action: The pool can be imported using its name or numeric identifier. 
 config:
	otus                         ONLINE 
	  mirror-0                   ONLINE 
	    /root/zpoolexport/filea  ONLINE 
	    /root/zpoolexport/fileb  ONLINE
```
### Сделаем импорт данного пула к нам в ОС:
> zpool import -d zpoolexport/ otus
### И проверим:
> zpool status
### Вывод команды:
```ruby
pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors  ....
```
> zpool get all otus
### Вывод команды:
```ruby
NAME  PROPERTY                       VALUE                          SOURCE 
otus  size                           480M                           - 
otus  capacity                       0%                             - 
otus  altroot                        -                              default 
otus  health                         ONLINE                         - 
otus  guid                           6554193320433390805            - 
otus  version                        -                              default 
otus  bootfs                         -                              default 
otus  delegation                     on                             default 
otus  autoreplace                    off                            default 
otus  cachefile                      -                              default 
otus  failmode                       wait                           default 
otus  listsnapshots                  off                            default 
otus  autoexpand                     off                            default 
otus  dedupditto                     0                              default 
otus  dedupratio                     1.00x                          - 
otus  free                           478M                           - 
otus  allocated                      2.09M                          - 
otus  readonly                       off                            - 
otus  ashift                         0                              default 
otus  comment                        -                              default 
otus  expandsize                     -                              - 
otus  freeing                        0                              - 
otus  fragmentation                  0%                             - 
otus  leaked                         0                              - 
otus  multihost                      off                            default 
otus  checkpoint                     -                              - 
otus  load_guid                      10988198376503386060           - 
otus  autotrim                       off                            default 
otus  feature@async_destroy          enabled                        local 
otus  feature@empty_bpobj            active                         local 
otus  feature@lz4_compress           active                         local 
otus  feature@multi_vdev_crash_dump  enabled                        local 
otus  feature@spacemap_histogram     active                         local 
otus  feature@enabled_txg            active                         local 
otus  feature@hole_birth             active                         local 
otus  feature@extensible_dataset     active                         local 
otus  feature@embedded_data          active                         local 
otus  feature@bookmarks              enabled                        local 
otus  feature@filesystem_limits      enabled                        local 
otus  feature@large_blocks           enabled                        local 
otus  feature@large_dnode            enabled                        local 
otus  feature@sha512                 enabled                        local 
otus  feature@skein                  enabled                        local 
otus  feature@edonr                  enabled                        local 
otus  feature@userobj_accounting     active                         local 
otus  feature@encryption             enabled                        local 
otus  feature@project_quota          active                         local 
otus  feature@device_removal         enabled                        local 
otus  feature@obsolete_counts        enabled                        local 
otus  feature@zpool_checkpoint       enabled                        local 
otus  feature@spacemap_v2            active                         local 
otus  feature@allocation_classes     enabled                        local 
otus  feature@resilver_defer         enabled                        local 
otus  feature@bookmark_v2            enabled                        local
```
### Запрос сразу всех параметром файловой системы:
> zfs get all otus
### Вывод команды:
```ruby
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```
### C помощью команды get можно уточнить конкретный параметр, например:
### Размер
> zfs get available otus
### Вывод команды:
```ruby
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```
### Тип:
> zfs get readonly otus
### Вывод команды:
```ruby
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```
### Значение recordsize:
> zfs get recordsize otus
### Вывод команды:
```ruby
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
### Тип сжатия (или параметр отключения):
> zfs get compression otus
### Вывод команды:
```ruby
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
### Тип контрольной суммы:
> zfs get checksum otus
### Вывод команды:
```ruby
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
## 3. Работа со снапшотом, поиск сообщения от преподавателя
### Скачаем файл, указанный в задании:
> wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
### Восстановим файловую систему из снапшота:
> zfs receive otus/test@today < otus_task2.file
### Далее, ищем в каталоге /otus/test файл с именем “secret_message”:
> find /otus/test -name "secret_message"
### Вывод команды:
```ruby
 /otus/test/task1/file_mess/secret_message
```
### Смотрим содержимое найденного файла:
> cat /otus/test/task1/file_mess/secret_message
### Вывод команды:
```ruby
https://otus.ru/lessons/linux-hl/
```
### Видим ссылку на курс OTUS, задание выполнено.
