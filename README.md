# Практические навыки работы с ZFS

## Цель:
- определить алгоритм с наилучшим сжатием;
- определить настройки pool’a;
- найти сообщение от преподавателей;
- составить список команд, которыми получен результат с их выводами.

### Определим алгоритм с наилучшим сжатием.

Смотрим список всех дисков:
```
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:128  0   40G  0 disk 
`-sda1   8:129  0   40G  0 part / 
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 
```
Создаём 4 пула, каждый из двух дисков, в режиме RAID 1:
```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
```
Добавим разные алгоритмы сжатия в каждую файловую систему:
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
Скачаем один и тот же текстовый файл во все пулы:
```
[root@zfs ~]# wget -O War_and_Peace.txt http://www.gutenberg.org/ebooks/2600.txt.utf-8
[root@zfs ~]# for i in {1..4}; do cp War_and_Peace.txt /otus$i; done
[root@zfs ~]# ls -l /otus*
/otus1:
total 2443
-rw-r--r--. 1 root root 3359630 Dec  6 07:13 War_and_Peace.txt

/otus2:
total 2041
-rw-r--r--. 1 root root 3359630 Dec  6 07:13 War_and_Peace.txt

/otus3:
total 1239
-rw-r--r--. 1 root root 3359630 Dec  6 07:13 War_and_Peace.txt

/otus4:
total 3287
-rw-r--r--. 1 root root 3359630 Dec  6 07:13 War_and_Peace.txt
```
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
```
[root@zfs ~]# zfs list                                              
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  2.49M   349M     2.41M  /otus1
otus2  2.10M   350M     2.02M  /otus2
otus3  1.32M   351M     1.23M  /otus3
otus4  3.32M   349M     3.23M  /otus4
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.35x                  -
otus2  compressratio         1.62x                  -
otus3  compressratio         2.64x                  -
otus4  compressratio         1.01x                  -
```
Мы видим, что алгоритм сжатия gzip-9 самый эффективный.

### Определение настроек пула.

Скачиваем архив в домашний каталог и разархивируем его:
```
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download' && tar -xzvf archive.tar.gz
...
2023-12-06 07:35:31 (2.59 MB/s) - 'archive.tar.gz' saved [7275140/7275140]

zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
Сделаем импорт данного пула к нам в ОС:
```
[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
```
Определяем настройки:
- Размер:
```
[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```
- Тип:
```
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```
- Значение recordsize:
```
[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
- Тип сжатия:
```
[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
- Тип контрольной суммы:
```
[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```

### Работа со снапшотом, поиск сообщения от преподавателя

Скачаем файл, указанный в задании:
```
[root@zfs ~]# wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download"
...
100%[===============================================>] 5,432,736   2.36MB/s   in 2.2s   

2023-12-06 07:57:24 (2.36 MB/s) - 'otus_task2.file' saved [5432736/5432736]

```
Восстановим файловую систему из снапшота и поищем в каталоге /otus/test файл с именем “secret_message”:
```
[root@zfs ~]# zfs receive otus/test@today < otus_task2.file
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```
\
В Vagrantfile добавлен скрипт для установки и конфигурации ZFS.
В итоге получим 4 пула по 2 диска в каждом(RAID-1), с разным сжатием.