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

### Способ 1. (`init=/bin/sh`)

* Переходим стрелкой к строке, начинающейся с `linux16`, нажимаем `End` чтобы перейти курсором в конец строки и клавишой `Backspace` удаляем всё до параметра `root=UUID=<UUID>` и после добавляем `init=/bin/sh` (пример на скриншоте ниже) нажимаем `сtrl-x` для загрузки в систему

![Screenshot1](https://raw.githubusercontent.com/mmmex/grub/master/Screenshot1.png)

* В целом на этом всё, Вы попали в систему. Но есть один нюанс. Рутовая файловая система при этом монтируется в режиме `Read-Only`. Если вы хотите перемонтировать ее в
режим `Read-Write` можно воспользоваться командой `mount -o remount,rw /`

* После чего можно убедится записав данные в любой файл или прочитав вывод команды: `mount | grep root`

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

* Первым делом посмотрим текущее состояние системы:

```sh
[root@otuslinux ~]# vgs
 VG #PV #LV #SN Attr VSize VFree
 VolGroup00 1 2 0 wz--n- <38.97g 0
```

* Нас интересует вторая строка с именем `Volume Group`

* Приступим к переименованию:

```sh
root@otuslinux ~]# vgrename VolGroup00 OtusRoot
Volume group "VolGroup00" successfully renamed to "OtusRoot
```

* Далее правим `/etc/fstab`, `/etc/default/grub`, `/boot/grub2/grub.cfg`. Везде заменяем старое
название на новое. По ссылкам можно увидеть примеры получившихся файлов.

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


