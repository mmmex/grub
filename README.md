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

![ScreenshotMenuEdit](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot_MenuEdit.png)

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

```sh
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

```sh
[vagrant@localhost ~]$ sudo -i
[root@localhost ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0
```

* Нас интересует строка с именем `VolGroup00`

* Приступим к переименованию:

```sh
[root@localhost ~]# vgrename VolGroup00 OtusRoot
  Volume group "VolGroup00" successfully renamed to "OtusRoot"
```

* Далее правим `/etc/fstab`, `/etc/default/grub`, `/boot/grub2/grub.cfg`. Везде заменяем старое название на новое. По ссылкам можно увидеть примеры получившихся файлов.

```sh
[root@localhost ~]# declare -a array=("/etc/default/grub" "/etc/fstab" "/boot/grub2/grub.cfg")
[root@localhost ~]# for chpath in "${array[@]}"; do
>     sed -i 's/VolGroup00/OtusRoot/g' $chpath
> done
```

<details><summary>Файл /etc/default/grub</summary>
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
</details>

<details><summary>Файл /etc/fstab</summary>
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0
</details>

<details><summary>Файл /boot/grub2/grub.cfg</summary>
#
# DO NOT EDIT THIS FILE
#
# It is automatically generated by grub2-mkconfig using templates
# from /etc/grub.d and settings from /etc/default/grub
#

### BEGIN /etc/grub.d/00_header ###
set pager=1

if [ -s $prefix/grubenv ]; then
  load_env
fi
if [ "${next_entry}" ] ; then
   set default="${next_entry}"
   set next_entry=
   save_env next_entry
   set boot_once=true
else
   set default="${saved_entry}"
fi

if [ x"${feature_menuentry_id}" = xy ]; then
  menuentry_id_option="--id"
else
  menuentry_id_option=""
fi

export menuentry_id_option

if [ "${prev_saved_entry}" ]; then
  set saved_entry="${prev_saved_entry}"
  save_env saved_entry
  set prev_saved_entry=
  save_env prev_saved_entry
  set boot_once=true
fi

function savedefault {
  if [ -z "${boot_once}" ]; then
    saved_entry="${chosen}"
    save_env saved_entry
  fi
}

function load_video {
  if [ x$feature_all_video_module = xy ]; then
    insmod all_video
  else
    insmod efi_gop
    insmod efi_uga
    insmod ieee1275_fb
    insmod vbe
    insmod vga
    insmod video_bochs
    insmod video_cirrus
  fi
}

terminal_output console
if [ x$feature_timeout_style = xy ] ; then
  set timeout_style=menu
  set timeout=1
# Fallback normal timeout code in case the timeout_style feature is
# unavailable.
else
  set timeout=1
fi
### END /etc/grub.d/00_header ###

### BEGIN /etc/grub.d/00_tuned ###
set tuned_params=""
set tuned_initrd=""
### END /etc/grub.d/00_tuned ###

### BEGIN /etc/grub.d/01_users ###
if [ -f ${prefix}/user.cfg ]; then
  source ${prefix}/user.cfg
  if [ -n "${GRUB2_PASSWORD}" ]; then
    set superusers="root"
    export superusers
    password_pbkdf2 root ${GRUB2_PASSWORD}
  fi
fi
### END /etc/grub.d/01_users ###

### BEGIN /etc/grub.d/10_linux ###
menuentry 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.2.3.el7.x86_64-advanced-b60e9498-0baa-4d9f-90aa-069048217fee' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_msdos
        insmod xfs
        set root='hd0,msdos2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint='hd0,msdos2'  570897ca-e759-4c81-90cf-389da6eee4cc
        else
          search --no-floppy --fs-uuid --set=root 570897ca-e759-4c81-90cf-389da6eee4cc
        fi
        linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/OtusRoot-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet
        initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img
}
if [ "x$default" = 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' ]; then default='Advanced options for CentOS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)'; fi;
### END /etc/grub.d/10_linux ###

### BEGIN /etc/grub.d/20_linux_xen ###
### END /etc/grub.d/20_linux_xen ###

### BEGIN /etc/grub.d/20_ppc_terminfo ###
### END /etc/grub.d/20_ppc_terminfo ###

### BEGIN /etc/grub.d/30_os-prober ###
### END /etc/grub.d/30_os-prober ###

### BEGIN /etc/grub.d/40_custom ###
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
### END /etc/grub.d/40_custom ###

### BEGIN /etc/grub.d/41_custom ###
if [ -f  ${config_directory}/custom.cfg ]; then
  source ${config_directory}/custom.cfg
elif [ -z "${config_directory}" -a -f  $prefix/custom.cfg ]; then
  source $prefix/custom.cfg;
fi
### END /etc/grub.d/41_custom ###
</details>

* Пересоздаем `initrd image`, чтобы он знал новое название `Volume Group`

```sh
[root@otuslinux ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
...
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

* После чего можем перезагружаться и если все сделано правильно успешно грузимся с
новым именем `Volume Group` и проверяем:

```sh
[root@otuslinux ~]# vgs
 VG #PV #LV #SN Attr VSize VFree
 OtusRoot 1 2 0 wz--n- <38.97g 0
```

* При желании можно так же заменить название `Logical Volume`

## Добавить модуль в `initrd`

Скрипты модулей хранятся в каталоге `/usr/lib/dracut/modules.d/`. Для того чтобы
добавить свой модуль создаем там папку с именем `01test`:

```[root@otuslinux ~]# mkdir /usr/lib/dracut/modules.d/01test```

В нее поместим два скрипта:

1. [module_setup.sh](/module_setup.sh) - который устанавливает модуль и вызывает скрипт `test.sh`
2. [test.sh](/test.sh) - собственно сам вызываемый скрипт, в нём у нас рисуется пингвинчик

* Пересобираем образ `initrd`

```root@otuslinux ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)```

или

```[root@otuslinux ~]# dracut -f -v```

* Можно проверить/посмотреть какие модули загружены в образ:

```sh
[root@otuslinux ~]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```

* После чего можно пойти двумя путями для проверки:
	* Перезагрузиться и руками выключить опции `rghb` и `quiet` и увидеть вывод
	* Либо отредактировать `grub.cfg` убрав эти опции

* В итоге при загрузке будет пауза на 10 секунд и вы увидите пингвина в выводе
терминала


