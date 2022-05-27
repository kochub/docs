---
sourcePath: ru/ydb/ydb-docs-core/ru/core/deploy/configuration/config.md
---
# Конфигурация кластера

Конфигурация кластера задается в YAML-файле, передаваемом в параметре `--yaml-config` при запуске узлов кластера.

В статье приведено описание основных групп конфигурируемых параметров в данном файле.

## host_configs - Типовые конфигурации хостов {#host-configs}

Кластер YDB состоит из множества узлов, для развертывания которых обычно используется одна или несколько типовых конфигураций серверов. Для того чтобы не повторять её описание для каждого узла, в файле конфигурации существует раздел `host_configs`, в котором перечислены используемые конфигурации, и им присвоены идентификаторы.

**Синтаксис**

``` yaml
host_configs:
- host_config_id: 1
  drive:
  - path: <path_to_device>
    type: <type>
  - path: ...
- host_config_id: 2
  ...  
```

Атрибут `host_config_id` задает числовой идентификатор конфигурации. В атрибуте `drive` содержится коллекция описаний подключенных дисков. Каждой описание состоит из двух атрибутов:

- `path` : Путь к смонтированному блочному устройству, например `/dev/disk/by-partlabel/ydb_disk_ssd_01`
- `type` : Тип физического носителя устройства: `ssd`, `nvme` или `rot` (rotational - HDD)

**Примеры**

Одна конфигурация с идентификатором 1, с одним диском типа SSD, доступным по пути `/dev/disk/by-partlabel/ydb_disk_ssd_01`:

``` yaml
host_configs:
- host_config_id: 1
  drive:
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_01
    type: SSD
```

Две конфигурации с идентификаторами 1 (два SSD диска) и 2 (три SSD диска):

``` yaml
host_configs:
- host_config_id: 1
  drive:
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_01
    type: SSD
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_02
    type: SSD
- host_config_id: 2
  drive:
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_01
    type: SSD
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_02
    type: SSD
  - path: /dev/disk/by-partlabel/ydb_disk_ssd_03
    type: SSD
```

### Особенности Kubernetes {#host-configs-k8s}

YDB Kubernetes operator монтирует NBS диски для Storage узлов на путь `/dev/kikimr_ssd_00`. Для их использования должна быть указана следующая конфигурация `host_configs`:

``` yaml
host_configs:
- host_config_id: 1
  drive:
  - path: /dev/kikimr_ssd_00
    type: SSD
```

Файлы с примерами конфигурации, поставляемые в составе YDB Kubernetes operator, уже содержат такую секцию, и её не нужно менять.

## hosts - Статические узлы кластера {#hosts}

В данной группе перечисляются статические узлы кластера, на которых запускаются процессы работы со Storage, и задаются их основные характеристики:

- Числовой индентификатор узла
- DNS-имя хоста и порт, по которым может быть установлено соединение с узлом в IP network
- Идентификатор [типовой конфигурации хоста](#host-configs)
- Размещение в определенной зоне доступности, стойке
- Инвентарный номер сервера (опционально)

**Синтаксис**

``` yaml
hosts:
- host: <DNS-имя хоста>
  host_config_id: <числовой идентификатор типовой конфигурации хоста>
  port: <порт> # 19001 по умолчанию
  location:
    unit: <строка с инвентарным номером сервера>
    data_center: <строка с идентификатором зоны доступности>
    rack: <строка с идентификатором стойки>
- host: <DNS-имя хоста>
  ...
```

**Примеры**

``` yaml
hosts:
- host: hostname1
  host_config_id: 1
  node_id: 1
  port: 19001
  location:
    unit: '1'
    data_center: '1'
    rack: '1'
- host: hostname2
  host_config_id: 1
  node_id: 2
  port: 19001
  location:
    unit: '1'
    data_center: '1'
    rack: '1'
```

### Особенности Kubernetes {#hosts-k8s}

При развертывании YDB с помощью оператора Kubernetes секция `hosts` полностью генерируется автоматически, заменяя любой указанный пользователем контент в передаваемой оператору конфигурации. Все Storage узлы используют `host_config_id` = `1`, для которого должна быть задана [корректная конфигурация](#host-configs-k8s).

## domains_config - Домен кластера {#domains-config}

Необходимо указать имя домена и типы хранилища, также необходимо указать номера хостов, которые будут входить `StateStorage` и параметр `nto_select`.
В конфигурации хранилища необходимо указать имя типа хранилища и тип отказоустойчивости хранилища (`erasure`), который будет использоваться для инициализации хранилища баз данных.
Также необходимо указать, каким типам дисков данный тип хранилища будет соответствовать. Доступны следующие схемы отказоустойчивости хранилища:

* Конфигурация `block-4-2` предполагает развертывание в 8 доменах отказа (по умолчанию доменом отказа является стойка) и выдерживает отказ не более чем 2 доменов отказа.
* Конфигурация `mirror-3-dc` предполагает развертывание в 3 ЦОД-ах, в каждом из которых располагается 3 домена отказоустойчивости и выдерживает отказ одного ЦОД-а и еще одного домена отказоустойчивости (стойки)
* Конфигурация `none` не предполагает отказоустойчивости, но удобна для функционального тестирования.

Отказоустойчивость `StateStorage` определяется параметром `nto_select`. Конфигурация `StateStorage` является отказоустойчивой, если любое подмножество из `nto_select` серверов, входящих в `StateStorage`, является отказоустойчивым.
`StateStorage` остается доступным, если для произвольного подмножества из `nto_select` серверов, входящих в `StateStorage`, доступно большинство из хостов.

Например, если в кластере используется модель отказоустойчивости `block-4-2` для хранилища, то чтобы `StateStorage` был отказоустойчивым, необходимо в качестве параметра  `nto_select` выбрать 5, а хосты разместить в 5 различных доменах доступности.
Если в кластере используется модель отказоустойчивости `mirror-3-dc`, то чтобы `StateStorage` был отказоустойчивым, необходимо в качестве параметра  `nto_select` выбрать 9, а хосты разместить в 3 ЦОД-ах, по 3 домена доступности в каждом ЦОД-е.

{% note warning %}

NToSelect должен быть нечетным. Для корректной работы `StateStorage` также необходимо, чтобы большинство из NToSelect реплик были доступны для произвольного набора из NToSelect хостов `StateStorage`.

{% endnote %}

Например, для функционального тестирования на одном сервере может использоваться данная конфигурация:

```bash
domains_config:
  domain:
  - name: Root
    storage_pool_types:
    - kind: ssd
      pool_config:
        box_id: 1
        erasure_species: none
        kind: ssd
        pdisk_filter:
        - property:
          - type: SSD
        vdisk_kind: Default
  state_storage:
  - ring:
      node:
      - 1
      nto_select: 1
    ssid: 1
```

В таком случае домен будет иметь имя `Root`, а в нем будет создан тип хранилища `ssd`. Данный тип хранилища будет соответствовать дискам, у которых параметр `type` будет равен значению `SSD`.
В параметре `erasure_species: none` означает, что хранилище будет создано без отказоустойчивости.

В случае, если кластер расположен в трех зонах доступности а в каждой из зон доступно по 3 сервера, то в таком случае конфигурация может такой:

```bash
domains_config:
  domain:
  - name: global
    storage_pool_types:
    - kind: ssd
      pool_config:
        box_id: 1
        erasure_species: mirror-3-dc
        kind: ssd
        pdisk_filter:
        - property:
          - type: SSD
        vdisk_kind: Default
  state_storage:
  - ring:
      node: [1, 2, 3, 4, 5, 6, 7, 8, 9]
      nto_select: 9
    ssid: 1
```

В таком случае домен будет иметь имя `global`, в нем также будет создан тип хранилища `ssd`. `erasure_species: mirror-3-dc` означает, что хранилище будет создано со схемой отказоустойчивости
`mirror-3-dc`. `StateStorage` будет растянут на 9 серверов с параметром `nto_select` равным 9.

## actor_system_config - Акторная система {#actor-system}

Создайте конфигурацию акторной системы. Необходимо указать распределение ядер процессора по пулам из числа доступных ядер в системе.

```bash
actor_system_config:
  executor:
  - name: System
    spin_threshold: 0
    threads: 2
    type: BASIC
  - name: User
    spin_threshold: 0
    threads: 3
    type: BASIC
  - name: Batch
    spin_threshold: 0
    threads: 2
    type: BASIC
  - name: IO
    threads: 1
    time_per_mailbox_micro_secs: 100
    type: IO
  - name: IC
    spin_threshold: 10
    threads: 1
    time_per_mailbox_micro_secs: 100
    type: BASIC
  scheduler:
    progress_threshold: 10000
    resolution: 256
    spin_threshold: 0
```

{% note warning %}

Не рекомендуется суммарно назначать в пулы IC, Batch, System, User больше ядер, чем доступно в системе.

{% endnote %}

## blob_storage_config - Статическая группа кластера {#blob-storage-config}

Укажите конфигурацию статической группы кластера. Статическая группа необходима для работы базовых таблеток кластера, в том числе `Hive`, `SchemeShard`, `BlobstorageContoller`.
Обычно данные таблетки не хранят много информации, поэтому мы не рекомендуем создавать более одной статической группы.

Для статической группы необходимо указать информацию о дисках и нодах, на которых будет размещена статическая группа. Например, для модели `erasure: none` конфигурация может быть такой:

```bash
blob_storage_config:
  service_set:
    groups:
    - erasure_species: none
      rings:
      - fail_domains:
        - vdisk_locations:
          - node_id: 1
            path: /dev/disk/by-partlabel/ydb_disk_ssd_02
            pdisk_category: SSD
....
```

Для конфигурации расположенной в 3 AZ  необходимо указать 3 кольца. Для конфигураций, расположенных в одной AZ, указывается ровно одно кольцо.

## Примеры конфигураций кластеров {#examples}

В [репозитории](https://github.com/ydb-platform/ydb/tree/main/ydb/deploy/yaml_config_examples/)  можно найти модельные примеры конфигураций кластеров для самостоятельного развертывания. Ознакомьтесь с ними перед развертыванием кластера.
