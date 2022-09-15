```
Домашнее задание
Работа с загрузчиком

Описание/Пошаговая инструкция выполнения домашнего задания:
1. Попасть в систему без пароля несколькими способами.
2. Установить систему с LVM, после чего переименовать VG.
3. Добавить модуль в initrd.
4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM
Репозиторий с пропатченым grub: https://yum.rumyantsev.com/centos/7/x86_64/
PV необходимо инициализировать с параметром --bootloaderareasize 1m

Критерии оценки:
Описать действия, описать разницу между методами получения шелла в процессе загрузки.
Где получится - используем script, где не получается - словами или копипастой описываем действия.
```
## Попадаем в систему несколькими способами. 

Для получения доступа необходимо открыть GUI VirtualBox (или другую систему виртуализации), запустить виртуальную машину и при выборе ядра для загрузки нажать `e` - в данном контексте `edit`. Попадаем в окно где мы можем изменить параметры загрузки:

![ScreenshotMenu](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot_MenuGrub.png)

![ScreenshotMenuEdit](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot_EditMode.png)

Для данного раздела инструкции собрана ВМ `grub` в конфигурации `Vagrantfile` с включенной опцией `gui`. Для испытания нижеперечисленных способов ввести команду: `vagrant up grub`.

### Способ 1. (`init=/bin/sh`)

* Переходим стрелкой к строке, начинающейся с `linux16`, нажимаем `End` чтобы перейти курсором в конец строки и клавишой `Backspace` удаляем всё до параметра `root=UUID=<UUID>` и после добавляем `init=/bin/sh` (пример на скриншоте ниже) нажимаем `сtrl-x` для загрузки в систему

![Screenshot1](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot1.png)

* В целом на этом всё, Вы попали в систему. Но есть один нюанс. Рутовая файловая система при этом монтируется в режиме `Read-Only`. Если вы хотите перемонтировать ее в режим `Read-Write` можно воспользоваться командой `mount -o remount,rw /`

* После чего можно убедится записав данные в любой файл или прочитав вывод команды: `touch /test_rw.txt`

### Способ 2. (`rd.break`)

* Переходим стрелкой к строке, начинающейся с `linux16`, нажимаем `End` чтобы перейти курсором в конец строки и клавишой `Backspace` удаляем всё до параметра `root=UUID=<UUID>` и после добавляем `rd.break` (пример на скриншоте ниже) нажимаем `сtrl-x` для загрузки в систему

![Screenshot2](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot2.png)

* Попадаем в `emergency mode` 

![ScreenshotEmergencyMode](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot_EmergencyMode.png)

* Корневая файловая система смонтирована в каталог `/sysroot` и в режиме `Read-Only`. Далее будет пример как попасть в нее и поменять пароль администратора:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
```

* После чего можно перезагружаться и заходить в систему с новым паролем. Полезно когда вы потеряли или вообще не имели пароль администратора.

### Способ 3. (`rw init=/sysroot/bin/sh`)

* Переходим стрелкой к строке, начинающейся с `linux16`, нажимаем `End` чтобы перейти курсором в конец строки и клавишой `Backspace` удаляем всё до параметра `root=UUID=<UUID>` и после добавляем `rw init=/sysroot/bin/sh` (пример на скриншоте ниже) нажимаем `сtrl-x` для загрузки в систему

![Screenshot3](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot3.png)

* В целом то же самое что и в прошлом примере, но файловая система сразу смонтирована в режим `Read-Write`

* В прошлых примерах тоже можно добавить `rw` после параметра `root=UUID=<UUID>`

## Установить систему с LVM, после чего переименовать VG

Для выполнения задания потребуется скопировать следующий [Vagrantfile](https://github.com/mmmex/homework3/blob/master/Vagrantfile)

* Запускаем ВМ `vagrant up`, переходим в консоль SSH `vagrant ssh` и выполняем `sudo -i`.

* Первым делом посмотрим текущее состояние системы:

```bash
[vagrant@localhost ~]$ sudo -i
[root@localhost ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
```

* Нас интересует строка с именем `VolGroup00`

* Приступим к переименованию:

```bash
[root@localhost ~]# vgrename VolGroup00 OtusRoot
  Volume group "VolGroup00" successfully renamed to "OtusRoot"
```

* Далее правим `/etc/fstab`, `/etc/default/grub`, `/boot/grub2/grub.cfg`. Везде заменяем старое название на новое. По ссылкам можно увидеть примеры получившихся файлов.

```bash
[root@localhost ~]# declare -a array=("/etc/default/grub" "/etc/fstab" "/boot/grub2/grub.cfg")
[root@localhost ~]# for chpath in "${array[@]}"; do
>     sed -i 's/VolGroup00/OtusRoot/g' $chpath
> done
```

[Файл /etc/default/grub](/grub)

[Файл /etc/fstab](/fstab)

[Файл /boot/grub2/grub.cfg](/grub.cfg)


* Пересоздаем `initrd image`, чтобы он знал новое название `Volume Group`

```bash
[root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

* После чего можем перезагружаться и если все сделано правильно успешно грузимся с новым именем `OtusRoot` и проверяем:

```bash
[root@localhost ~]# vgs
  VG       #PV #LV #SN Attr   VSize   VFree
  OtusRoot   1   2   0 wz--n- <38.97g    0
```

* При желании можно так же заменить название `Logical Volume`

## Добавить модуль в `initrd`

Будем использовать `Vagrantfile` из предыдущей инструкции, но для удобства добавим свойство `vb.gui = true` после строки `box.vm.provider :virtualbox do |vb|`. Перечитаем конфигурацию командой `vagrant reload`.

Скрипты модулей хранятся в каталоге `/usr/lib/dracut/modules.d/`. Для того чтобы добавить свой модуль создаем там папку с именем `01test`:

```[root@localhost ~]# mkdir /usr/lib/dracut/modules.d/01test```

В нее поместим два скрипта:

1. [module-setup.sh](/module-setup.sh) - который устанавливает модуль и вызывает скрипт `test.sh`
2. [test.sh](/test.sh) - собственно сам вызываемый скрипт, в нём у нас рисуется пингвинчик

Переходим в каталог и выставляем флаг `x`:

```bash
[root@localhost ~]# cd /usr/lib/dracut/modules.d/01test
[root@localhost 01test]# chmod a+x *
[root@localhost 01test]# ls -al
total 12
drwxr-xr-x.  2 root root   44 Sep 15 23:03 .
drwxr-xr-x. 52 root root 4096 Sep 15 22:58 ..
-rwxr-xr-x.  1 root root  111 Sep 15 23:03 module-setup.sh
-rwxr-xr-x.  1 root root  316 Sep 15 23:03 test.sh
```

* Пересобираем образ `initrd`

```bash
[root@localhost 01test]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: test ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

или

```[root@localhost 01test]# dracut -f```

* Можно проверить/посмотреть какие модули загружены в образ:

```sh
[root@localhost 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```

* После чего можно пойти двумя путями для проверки:
	* Перезагрузиться и руками выключить опции `rghb` и `quiet` и увидеть вывод
	* Либо отредактировать `grub.cfg` убрав эти опции

* В итоге при загрузке будет пауза на 10 секунд и вы увидите пингвина в выводе терминала

![ScreenshotSracut](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot_Dracut.png)
