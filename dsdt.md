#Памятка по работе с DSDT

DSDT (Differentiated System Description Table) является частью стандарта ACPI, содержит в себе информацию о системе питания.

Получить текущую таблицу можно следующим способом:
```sh
# cat /sys/firmware/acpi/tables/DSDT > dsdt.dat
```

Для модификации таблицы её необходимо дизассемблировать:
```sh
$ iasl -d dsdt.dat
```

На этом месте у меня начались проблемы. Компилятор упорно не хочет собирать таблицу из выхлопа своего же дизассемблера. Поэтому для теста таблица заменяется на саму себя.

Для подмены таблицы через GRUB2, необходимо пересобрать его конфиг. Для начала, создадим файл /etc/grub.d/01_acpi со следующим содержимым:
```sh
#! /bin/sh -e

# Uncomment to load custom ACPI table
GRUB_CUSTOM_ACPI="/boot/dsdt.aml"

# DON'T MODIFY ANYTHING BELOW THIS LINE!

libdir=/usr/share

. ${libdir}/grub/grub-mkconfig_lib

# Load custom ACPI table
if [ x${GRUB_CUSTOM_ACPI} != x ] && [ -f ${GRUB_CUSTOM_ACPI} ] \
        && is_path_readable_by_grub ${GRUB_CUSTOM_ACPI}; then
    echo "Found custom ACPI table: ${GRUB_CUSTOM_ACPI}" >&2
    prepare_grub_to_access_device `${grub_probe} --target=device ${GRUB_CUSTOM_ACPI}` | sed -e "s/^/  /"
    cat << EOF
acpi (\$root)`make_system_path_relative_to_its_root ${GRUB_CUSTOM_ACPI}`
EOF
fi
```
В скрипте присутствует переменная libdir, проверьте, есть ли нужные файлы по этому пути. Также проверьте путь к кастомной таблице dsdt.


Далее необходимо сделать данный файл исполняемым:
```sh
# chmod a+x /etc/grub.d/01_acpi
```


На этом с конфигом вроде бы все. После этого запускаем
```sh
# grub-mkconfig -o /boot/grub/grub.cfg
Генерируется файл настройки grub …
Found custom ACPI table: /boot/dsdt.aml             <------- Вот то, что нам нужно
Найден образ linux: /boot/vmlinuz-linux
Найден образ initrd: /boot/intel-ucode.img /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  intel-ucode.img initramfs-linux-fallback.img
Предупреждение: os-prober не будет запущен для обнаружения других загрузочных разделов.
Их системы не будут добавлены в загрузочные настройки GRUB.
Прочтите документацию на параметр GRUB_DISABLE_OS_PROBER.
Добавляется элемент загрузочного меню для настроек микропрограммы UEFI …
завершено
```


После модификации таблицы через GRUB2, в системном журнале появились следующие записи (на самом деле ничего не поменялось, так как файл тот же самый):
```sh
ACPI: DSDT 0x00000000BE4CA000 023590 (v02 ALASKA A M I    01072009 INTL 20120913)
ACPI: Reserving DSDT table memory at [mem 0xbe4ca000-0xbe4ed58f]
```
