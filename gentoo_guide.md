# Установка Gentoo Linux + Arch + LiveCD + Haskell

Так уж сложилось, что я программирую на Haskell. И к сожалению нет ни одного LiveCD с готовой средой разработки на Haskell.
Поэтому я сделал свой собственный LiveCD. В качестве операционных систем я использую Gentoo Linux  и Arch Linux. Помимо создания LiveCD,  я подробно опишу процесс установки обеих операционных систем и затем мы приступим к их настройке и созданию LiveCD.

Это руководство написано для тех, кто хочет подробно разобраться в установке Gentoo Linux и  Arch Linux. Написано очень много замечательный программ, так что все и не охватить и существует огромное количество способов, сделать то или иное действие. В этом руководстве я постараюсь освятить подробно установку Gentoo + Arch, так как ее вижу я. И первый вопрос, почему Gentoo + Arch. Можно ведь было описать отдельно процесс установки Gentoo и отдельно процесс установки Arch. Это руководство именно о том, как использовать обе операционные системы одновременно. Приведу скриншот, того о чем идет речь.

![Gentoo Screenshots](https://github.com/hardentoo/gentoo_guide/blob/master/2017-11-23-073554_1920x1080_scrot.png)

Конечный результат должен быть таким

* установленный и настроенный Gentoo Linux (x86_64)
* установленный и настроенный Arch Linux (x86_64)
* возможность загрузить любую из операционок как xen dom0
* возможность загрузить вторую из операционок как xen domU
* возможность загрузить любую из операционок под lxc
* возможность одновремменно использовать программы из gentoo и arch
* установленная и настроенная среда разработки на haskell
* созданный LiveCD

Замечательно, теперь когда определелились с целями, давайте опишем как все это у нас будет работать.

* UEFI -> GRUB -> Xen -> Dom0 (Gentoo) -> LXC (Arch)

Почему в этой схеме Dom0 -> LXC. На самом деле работать будут все варианты. Я же привел тот, который использую чаще всего.
Вот другие

* UEFI -> GRUB -> Xen -> Dom0 (Arch) -> LXC (Gentoo)
* UEFI -> GRUB -> Xen -> Dom0 (Gentoo) -> DomU (Arch)
* UEFI -> GRUB -> Xen -> Dom0 (Arch) -> DomU (Gentoo)

Наша установка будет соответствовать всем этим вариантам одновременно. Устанавливать будем не спеша, по ходу дела настраивая системы. Опишем чуть больше подробностей.

* UEFI -> GRUB
   * grub версии 2, соберем его с поддержкой Crypto Luks и ZFS
* GRUB -> XEN
   * grub дешифрует корневую файловую систему, на которой расположено все (gentoo --- /boot + /  + /home и  arch --- /boot + / + /home)
   * файловая система ZFS. Почему ZFS? Посмотрите список [плюшек](https://wiki.gentoo.org/wiki/ZFS/Features)
   * грузится xen
   * будем по возможности использовать hardened профили и версии программ
* Xen -> Dom0
   * Если грузится gentoo dom0, то xen грузится тот который установлен в gentoo
   * Если грузится arch dom0, то xen грузится тот который установлен в arch

* После загрузки dom0, вы попадаете в соответствующую операционку и запускаете если хотите по желанию
   * либо xen domu со второй операционкой
   * либо lxc со второй операционкой

То есть у нас есть ровно 2 операционки, каждая из которых может быть загружена 3 способами, либо на физической машине, либо под xen, либо под lxc. Главное можно работать как в gentoo, так и в arch одноврмененно.

Теперь опишу как я буду настраивать среду разработки на haskell

* горячие клавиши мы повесим на все используя actkbd + xbindkeys (собранный с guile)
* менеджер окон --- мы будем использовать dwm (+ пачти) + dmenu (+ скрипты)
* терминал urxvt + скрипты на perl
* tmux + tmuxinator + плагины
* zsh (+zgen + плагины)
* vim + плагины

Собственно haskell
* ghc
* stack
* hoogle
* idris
* ...

Ну что поехали!

### Лучший способ стаить gentoo из под gentoo

Итак, чтобы установить gentoo нам необходима gentoo!

#### Подготовка диска и установка нужных пакетов

* добавим в файл /etc/portage/package.accept_keywords следующие строки

app-misc/pax-utils ~amd64
sys-kernel/genkernel ~amd64
sys-kernel/spl ~amd64
sys-fs/zfs-kmod ~amd64
sys-fs/zfs ~amd64

Здесь

app-misc/pax-utils --- ELF utils that can check files for security relevant properties
sys-kernel/genkernel --- Gentoo automatic kernel building scripts
sys-kernel/spl --- The Solaris Porting Layer is a Linux kernel module which provides many of the Solaris kernel APIs
sys-fs/zfs-kmod --- Linux ZFS kernel module for sys-fs/zfs
sys-fs/zfs --- Userland utilities for ZFS Linux kernel module

Если portage ругается на блокироку zfs и genkernel, значит надо поставить genkernel новее, так как он понимает zfs с определенной версии. Именно поэтому мы добавили ~amd64 для genkernel в
package.accept_keywords.

#### Установка xen

```bash
emerge xen xen-tools
```

```bash
grub2-install --modules="linux crypto search_fs_uuid luks lvm" --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=gentoo --recheck --debug
```

### Готовим раздел для установки

#### Создание раздела

Загрузим необходимые модули ядра, если они еще не загружены. Делаем

```bash
Gentoo# modprobe zfs
```

Смотрим с помощью lsmod и dmesg | tail -50, что все удачно.

```bash
Gentoo# lsmod
Module                  Size  Used by
zfs                  3317760  0
zunicode              331776  1 zfs
zavl                   16384  1 zfs
icp                   245760  1 zfs
zcommon                65536  1 zfs
znvpair                73728  2 zcommon,zfs
spl                    98304  4 znvpair,zcommon,zfs,icp
```

```bash
Gentoo# dmesg | tail -50
[123818.135472] SPL: Loaded module v0.7.3-r0-gentoo
[123818.138536] znvpair: module license 'CDDL' taints kernel.
[123818.138539] Disabling lock debugging due to kernel taint
[123820.052112] device: 'zfs': device_add
[123820.052175] PM: Adding info for No Bus:zfs
[123820.052547] ZFS: Loaded module v0.7.3-r0-gentoo, ZFS pool version 5000, ZFS filesystem version 5
```

Давайте посмотрим текущее разбиение диска

![Gentoo Screenshots](https://github.com/hardentoo/gentoo_guide/blob/master/2017-11-25-193528_1920x1080_scrot.png)

Мы удалим разделы /dev/sda5, /dev/sda6, /dev/sda7, и в получившемся свободном пространстве создадим раздел. Но предварительно, мы в файле /etc/conf.d/dmcrypt закомментируем строки, создающие swap раздел, так как нумерация gpt разделов изменится и мы не хотим потерять данные.

![Gentoo Screenshots](https://github.com/hardentoo/gentoo_guide/blob/master/2017-11-25-194856_1920x1080_scrot.png)

Теперь создадим раздел

![Gentoo Screenshots](https://github.com/hardentoo/gentoo_guide/blob/master/2017-11-25-204838_1920x1080_scrot.png)

#### Зачистка раздела

Для зачистки можно использовать wipe и shred. Мы используем shred. Чтоб ничего не потерять настоятельно рекомендую прочитать man shred. Чтобы узнать сколько это заняло времени, напишем перед кодмандой time.

```bash
Gentoo gentoo_guide # time shred -vfz -n 10 /dev/sda5
shred: /dev/sda5: проход 1/11 (random)…
shred: /dev/sda5: проход 1/11 (random)…331MiB/98GiB 0%
shred: /dev/sda5: проход 1/11 (random)…659MiB/98GiB 0%
```

Здесь

 * -v &mdash; показывать прогресс операции
 * -f &mdash; изменить права доступа если необходимо
 * -z &mdash; в конце заполнить нулями, чтобы спрятать shredding
 * -n &mdash; проделать N раз вместо 3 по умолчанию

![Gentoo Screenshots](https://github.com/hardentoo/gentoo_guide/blob/master/2017-11-26-124630_1920x1080_scrot.png)

#### Шифрование раздела

Чтобы выбрать какие алгоритмы использовать, попробуйте сделать 

```bash
cryptsetup benchmark
```

Я выбрал следующие

```bash
cryptsetup luksFormat /dev/sda5 --cipher=aes-cbc-essiv:sha256 --key-size=256 --hash=sha256
```

Посмотрим, что все хорошо

```
cryptsetup luksDump /dev/sda5 
```

Теперь добавим в один из слотов дополнительные ключи дешифровки. Вначале сгенерим ключ и положим его в файл /etc/keys/luks_key_sda5.key

```bash
dd if=/dev/urandom of=/etc/keys/luks_key_sda5.key bs=1 count=4096

```

Теперь добавим его к нашему разделу

```bash
cryptsetup luksAddKey /dev/sda5 /etc/keys/luks_key_sda5.key
```

Посмотрим, что все хорошо

```
cryptsetup luksDump /dev/sda5 
```

Увидим, что Key Slot 0: ENABLED и Key Slot 1: ENABLED. Теперь установим права и аттрибуты

```bash
cd /etc/keys/
chmod 0400 luks_key_sda5.key
chattr +i luks_key_sda5.key
```

Проверим, что все хорошо ls -l и lsattr. Подключаем раздел

```bash
cryptsetup luksOpen /dev/sda5 sda5crypt --key-file=/etc/keys/luks_key_sda5.key 
```

Ура, у нас теперь есть /dev/mapper/sda5crypt. Чтобы в дальнейшем ничего не перепутать (помимо sda5 у нас есть еще другие sdaN), создадим symlink, который и будем дальше использовать.

```
ln -s /dev/mapper/sda5crypt /dev/gentoo
```
