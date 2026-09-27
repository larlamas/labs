# Лабораторная работа №1
## Аудит, тюнинг и эксплуатация bare-metal хоста KVM

**Дисциплина:** Технологии виртуализации  
**ОС хоста:** Ubuntu Server 24.04 LTS  
**Гипервизор внешнего уровня:** Oracle VirtualBox 7.2.20  
**Процессор физического компьютера:** AMD Ryzen 7 9800X3D  
**Архитектура:** x86_64  

---

## 1. Цель работы

Цель работы - выполнить аудит аппаратных возможностей виртуализации x86-64, настроить Linux-хост для работы с KVM, активировать IOMMU, настроить статические HugePages, отключить THP и KSM, запустить гостевую ОС через QEMU/KVM и выполнить привязку потоков vCPU к отдельным процессорам хоста.

Лабораторная работа выполнялась во вложенной среде:

```text
AMD Ryzen 7 9800X3D
        ↓
Windows 11
        ↓
Oracle VirtualBox 7.2.20
        ↓
Ubuntu Server 24.04
        ↓
KVM/QEMU
        ↓
CirrOS 0.6.2
```

Для виртуальной машины VirtualBox была включена Nested Virtualization.

---

## 2. Задание 1. Аудит процессора и NUMA

### 2.1 Проверка аппаратной виртуализации

Выполнены команды:

```bash
grep -E -c '(vmx|svm)' /proc/cpuinfo
grep -E -o '(vmx|svm|ept|npt|tpr_shadow|vnmi|vpid)' /proc/cpuinfo | sort -u
```

Получен результат:

```text
4
ept
svm
```

Наличие флага `svm` подтверждает доступность AMD-V для гостевой Ubuntu.

Проверка KVM:

```bash
kvm-ok
```

Результат:

```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

Полные результаты сохранены в:

```text
logs/kvm-ok.txt
logs/virt-count.txt
logs/virt-flags.txt
```

### 2.2 Топология процессора

Команда:

```bash
lscpu
```

Основные параметры:

```text
Architecture:        x86_64
CPU(s):              4
Vendor ID:           AuthenticAMD
Model name:          AMD Ryzen 7 9800X3D 8-Core Processor
Thread(s) per core:  1
Core(s) per socket:  4
Socket(s):           1
Virtualization:      AMD-V
Hypervisor vendor:   KVM
Virtualization type: full
NUMA node(s):        1
NUMA node0 CPU(s):   0-3
```

| Параметр | Значение |
|---|---:|
| Сокетов | 1 |
| Ядер | 4 |
| Потоков на ядро | 1 |
| Логических CPU | 4 |
| NUMA-узлов | 1 |

Полный вывод находится в `logs/lscpu.txt`.

### 2.3 Анализ NUMA

Команда:

```bash
numactl --hardware
```

Результат:

```text
available: 1 nodes (0)
node 0 cpus: 0 1 2 3
node 0 size: 7934 MB
node 0 free: 4916 MB
node distances:
node   0
  0:  10
```

NUMA-узел в системе один: в него входят CPU 0–3 и около 7.9 ГБ оперативной памяти.

Полный вывод: `logs/numa.txt`.

---

## 3. Задание 2. Установка KVM и настройка IOMMU

Был установлен стек виртуализации:

```bash
sudo apt-get install --no-install-recommends -y qemu-system-x86 qemu-utils libvirt-daemon-system libvirt-clients
```

Проверка модулей KVM:

```bash
lsmod | grep kvm
```

На AMD-платформе были загружены модули `kvm_amd` и `kvm`.

### 3.1 IOMMU

В командной строке ядра используется параметр:

```text
iommu=pt
```

Проверка:

```bash
sudo dmesg | grep -i -E 'amd-vi|iommu|dmar'
```

Основные строки:

```text
AMD-Vi: Using global IVHD EFR:0x0, EFR2:0x0
iommu: Default domain type: Passthrough (set via kernel command line)
pci 0000:00:02.0: Adding to iommu group 0
pci 0000:00:03.0: Adding to iommu group 1
pci 0000:00:04.0: Adding to iommu group 2
pci 0000:00:05.0: Adding to iommu group 3
pci 0000:00:07.0: Adding to iommu group 4
AMD-Vi: Interrupt remapping enabled
```

По этим строкам видно, что AMD IOMMU инициализирован, а устройства распределены по IOMMU-группам.

Полный вывод: `logs/iommu.txt`.

### Примечание

В методических указаниях для AMD приведён параметр:

```text
amd_iommu=on iommu=pt
```

На используемом ядре параметр `amd_iommu=on` приводил к сообщению:

```text
AMD-Vi: Unknown option - 'on'
```

Поэтому в итоговой конфигурации остался только `iommu=pt`. AMD-Vi при этом всё равно инициализируется, и IOMMU работает - это видно по выводу `dmesg` выше.

---

## 4. Задание 3. HugePages, THP и KSM

### 4.1 Отключение THP

Выполнены команды:

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

Итоговое состояние:

```text
=== THP enabled ===
always madvise [never]

=== THP defrag ===
always defer defer+madvise madvise [never]
```

THP отключён.

### 4.2 Отключение KSM

Команда:

```bash
echo 0 | sudo tee /sys/kernel/mm/ksm/run
```

Результат:

```text
=== KSM ===
0
```

KSM отключён.

### 4.3 Статические HugePages

В `/etc/sysctl.d/99-kvm.conf` настроено:

```text
vm.nr_hugepages = 1024
vm.swappiness = 0
```

Размер одной HugePage:

```text
Hugepagesize: 2048 kB
```

Итого зарезервировано:

```text
1024 × 2 MiB = 2048 MiB
```

Полная проверка параметров: `logs/memory-settings.txt`.

---

## 5. Задание 4. Запуск CirrOS через QEMU/KVM

Использован образ:

```text
cirros-0.6.2-x86_64-disk.img
```

Проверка образа:

```bash
qemu-img info cirros-0.6.2-x86_64-disk.img
```

Основные параметры:

```text
file format: qcow2
virtual size: 112 MiB
disk size: 20.4 MiB
corrupt: false
```

### 5.1 Особенность Nested Virtualization

Первоначально применялся параметр:

```bash
-cpu host
```

Но при запуске вложенной ВМ внешний VirtualBox аварийно завершал работу с ошибкой:

```text
Guru Meditation -4060
VERR_SVM_UNKNOWN_EXIT
```

В ходе диагностики ВМ удалось запустить в следующих конфигурациях:

```text
qemu64 + 1 vCPU + 512 MB
qemu64 + 2 vCPU + 512 MB
qemu64 + 2 vCPU + 1024 MB
```

Поэтому для выполнения лабораторной использовалась модель CPU:

```bash
-cpu qemu64
```

### 5.2 Итоговая команда запуска

```bash
sudo qemu-system-x86_64 -enable-kvm -cpu qemu64 -smp 2 -m 1024 -mem-path /dev/hugepages -mem-prealloc -drive file=cirros-0.6.2-x86_64-disk.img,format=qcow2,if=virtio -nographic
```

Она сохранена в `qemu-command.txt`.

CirrOS загрузился и определил:

```text
Platform: QEMU Ubuntu 24.04 PC
Arch: x86_64
CPU(s): 2
Cores/Sockets/Threads: 2/1/1
Virt-type: AMD-V
RAM Size: 969MB
Hypervisor detected: KVM
```

---

## 6. Фактическое использование HugePages

### 6.1 До запуска QEMU

```text
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:         2097152 kB
```

Файл: `logs/hugepages-without-qemu.txt`.

### 6.2 При запущенной ВМ

```text
HugePages_Total:    1024
HugePages_Free:      512
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:         2097152 kB
```

Файл: `logs/hugepages-with-qemu.txt`.

### 6.3 Сравнение

| Параметр | Без QEMU | С QEMU |
|---|---:|---:|
| HugePages_Total | 1024 | 1024 |
| HugePages_Free | 1024 | 512 |
| HugePages_Rsvd | 0 | 0 |
| Hugepagesize | 2048 kB | 2048 kB |

Количество занятых страниц:

```text
1024 - 512 = 512 HugePages
```

Объём памяти:

```text
512 × 2 MiB = 1024 MiB
```

Значит, QEMU взял для гостя 1 ГБ из статического пула HugePages.

---

## 7. Профилирование потоков QEMU

PID процесса QEMU:

```text
3300
```

Команда:

```bash
ps -T -p 3300 -o pid,tid,psr,comm
```

Получены потоки:

```text
PID     TID  PSR  COMMAND
3300    3300  0   qemu-system-x86
3300    3301  3   qemu-system-x86
3300    3306  1   qemu-system-x86
3300    3307  2   qemu-system-x86
3300    3310  1   kvm-nx-lpage-re
```

Для vCPU использованы TID:

```text
3306
3307
```

Полный вывод: `logs/qemu-threads.txt`.

---

## 8. vCPU Pinning

До привязки:

```text
pid 3306's current affinity list: 0-3
pid 3307's current affinity list: 0-3
```

Выполнено:

```bash
sudo taskset -cp 1 3306
sudo taskset -cp 2 3307
```

Результат:

```text
pid 3306's new affinity list: 1
pid 3307's new affinity list: 2
```

Проверка:

```text
pid 3306's current affinity list: 1
pid 3307's current affinity list: 2
```

Итог:

```text
vCPU 0 (TID 3306) → CPU 1
vCPU 1 (TID 3307) → CPU 2
```

Логи:

```text
logs/vcpu0-affinity.txt
logs/vcpu1-affinity.txt
```

---

## 9. Контрольные точки

### KVM

```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

### IOMMU

```text
iommu: Default domain type: Passthrough
AMD-Vi: Interrupt remapping enabled
```

### HugePages

```text
До запуска ВМ: HugePages_Free = 1024
При работе ВМ: HugePages_Free = 512
```

Использовано:

```text
512 × 2 MiB = 1024 MiB
```

### vCPU Pinning

```text
TID 3306 → CPU 1
TID 3307 → CPU 2
```

---

## 10. Вывод

Аудит хоста показал, что AMD-V и KVM доступны.

Был установлен стек QEMU/KVM, IOMMU работает в режиме passthrough: система формирует IOMMU-группы, interrupt remapping включён.

Для подсистемы памяти создан статический пул из 1024 HugePages размером 2 МБ, то есть 2 ГБ зарезервированной памяти. Transparent HugePages и KSM отключены, `vm.swappiness` установлен в 0.

Гостевая система CirrOS 0.6.2 запущена через QEMU с аппаратным ускорением KVM и памятью из `/dev/hugepages`. Гостю выделен 1 ГБ оперативной памяти, и число свободных HugePages уменьшилось с 1024 до 512 - ВМ заняла 512 страниц по 2 МБ.

Потоки двух vCPU вручную привязаны к отдельным процессорам хоста:

```text
TID 3306 → CPU 1
TID 3307 → CPU 2
```

Основные задачи лабораторной работы выполнены. Отступить от методических указаний пришлось в двух местах, оба связаны с вложенной средой: вместо `amd_iommu=on iommu=pt` используется только `iommu=pt`, а вместо `-cpu host` - модель `-cpu qemu64`.

---

## 11. Файлы лабораторной работы

```text
kvm-lab1/
├── README.md
├── qemu-command.txt
├── cirros-0.6.2-x86_64-disk.img
└── logs/
    ├── hugepages-with-qemu.txt
    ├── hugepages-without-qemu.txt
    ├── iommu.txt
    ├── kvm-ok.txt
    ├── lscpu.txt
    ├── memory-settings.txt
    ├── numa.txt
    ├── qemu-threads.txt
    ├── vcpu0-affinity.txt
    ├── vcpu1-affinity.txt
    ├── virt-count.txt
    └── virt-flags.txt
```
