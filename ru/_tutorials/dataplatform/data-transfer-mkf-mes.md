# Поставка данных из очереди {{ KF }} {{ ES }} с помощью {{ data-transfer-full-name }}

{% include [Elasticsearch-end-of-service](../../_includes/mdb/mes/note-end-of-service.md) %}

В кластер {{ mes-name }} можно в реальном времени поставлять данные из топиков {{ KF }}.

Чтобы запустить поставку данных:

1. [Подготовьте тестовые данные](#prepare-data).
1. [Настройте кластер-приемник](#configure-target).
1. [Подготовьте и активируйте трансфер](#prepare-transfer).
1. [Проверьте работоспособность трансфера](#verify-transfer).

Если созданные ресурсы вам больше не нужны, [удалите их](#clear-out).

## Перед началом работы {#before-you-begin}

1. Подготовьте инфраструктуру поставки данных:

    {% list tabs %}

    - Вручную

        1. [Создайте кластер-источник {{ mkf-name }}](../../managed-kafka/operations/cluster-create.md) любой подходящей конфигурации. Для подключения к кластеру с локальной машины пользователя, а не из облачной сети {{ yandex-cloud }}, включите публичный доступ к кластеру при его создании.

        1. [Создайте в кластере-источнике топик](../../managed-kafka/operations/cluster-topics.md#create-topic) с именем `sensors`.

        1. [Создайте в кластере-источнике пользователя](../../managed-kafka/operations/cluster-accounts.md#create-user) с именем `mkf-user` и правами доступа `ACCESS_ROLE_PRODUCER` и `ACCESS_ROLE_CONSUMER` к созданному топику.

        1. [Создайте кластер-приемник {{ mes-name }}](../../managed-elasticsearch/operations/cluster-create.md) любой подходящей конфигурации со следующими настройками:

            * В той же зоне доступности, что и кластер-источник.
            * Для подключения к кластеру с локальной машины пользователя, а не из облачной сети {{ yandex-cloud }}, включите публичный доступ к хостам кластера.

        
        1. Для подключения к кластерам с локальной машины пользователя, настройте группы безопасности:

            * [{{ mkf-name }}](../../managed-kafka/operations/connect.md#configuring-security-groups).
            * [{{ mes-name }}](../../managed-elasticsearch/operations/cluster-connect.md#configuring-security-groups).

            {% include [preview-pp.md](../../_includes/preview-pp.md) %}


    - С помощью {{ TF }}

        1. Если у вас еще нет {{ TF }}, [установите и настройте его](../../tutorials/infrastructure-management/terraform-quickstart.md#install-terraform).
        1. Скачайте [файл с настройками провайдера](https://github.com/yandex-cloud/examples/tree/master/tutorials/terraform/provider.tf). Поместите его в отдельную рабочую директорию и [укажите значения параметров](../../tutorials/infrastructure-management/terraform-quickstart.md#configure-provider).
        1. Скачайте в ту же рабочую директорию файл конфигурации [data-transfer-mkf-mes.tf](https://github.com/yandex-cloud/examples/tree/master/tutorials/terraform/data-transfer/data-transfer-mkf-mes.tf).

            В этом файле описаны:

            * [сеть](../../vpc/concepts/network.md#network);
            * [подсеть](../../vpc/concepts/network.md#subnet);
            * [группа безопасности](../../vpc/concepts/security-groups.md) и правила, необходимые для подключения к кластерам {{ mkf-name }} и {{ mes-name }};
            * кластер-источник {{ mkf-name }};
            * топик {{ KF }} с именем `sensors`;
            * пользователь {{ KF }} `mkf-user` с правами доступа `ACCESS_ROLE_PRODUCER`, `ACCESS_ROLE_CONSUMER` к топику `sensors`;
            * кластер-приемник {{ mes-name }};
            * трансфер.

        1. Укажите в файле `data-transfer-mkf-mes.tf` переменные:

            * `source_kf_version` — версия {{ KF }} в кластере-источнике;
            * `source_user_password` — пароль пользователя `mkf-user` в кластере-источнике;
            * `target_admin_password` — пароль администратора кластера-приемника {{ mes-name }};
            * `transfer_enabled` — значение `0`, чтобы не создавать трансфер до [создания эндпоинтов вручную](#prepare-transfer).

        1. Выполните команду `terraform init` в директории с конфигурационным файлом. Эта команда инициализирует провайдер, указанный в конфигурационных файлах, и позволяет работать с ресурсами и источниками данных провайдера.
        1. Проверьте корректность файлов конфигурации {{ TF }} с помощью команды:

            ```bash
            terraform validate
            ```

            Если в файлах конфигурации есть ошибки, {{ TF }} на них укажет.

        1. Создайте необходимую инфраструктуру:

            {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

            {% include [explore-resources](../../_includes/mdb/terraform/explore-resources.md) %}

    {% endlist %}

1. Установите утилиты:

    - [kafkacat](https://github.com/edenhill/kcat) — для чтения и записи данных в топики {{ KF }}.

        ```bash
        sudo apt update && sudo apt install --yes kafkacat
        ```

        Убедитесь, что можете с ее помощью [подключиться к кластеру-источнику {{ mkf-name }} через SSL](../../managed-kafka/operations/connect.md#connection-string#bash).

    - [jq](https://stedolan.github.io/jq/) — для потоковой обработки JSON-файлов.

        ```bash
        sudo apt update && sudo apt-get install --yes jq
        ```

## Подготовьте тестовые данные {#prepare-data}

Пусть в качестве сообщения в топик {{ KF }} `sensors` кластера-источника поступают данные от сенсоров автомобиля в формате JSON.

Создайте локально файл `sample.json` с тестовыми данными:

{% cut "sample.json" %}

```json
{
    "device_id": "iv9a94th6rztooxh5ur2",
    "datetime": "2020-06-05 17:27:00",
    "latitude": 55.70329032,
    "longitude": 37.65472196,
    "altitude": 427.5,
    "speed": 0,
    "battery_voltage": 23.5,
    "cabin_temperature": 17,
    "fuel_level": null
}
{
    "device_id": "rhibbh3y08qmz3sdbrbu",
    "datetime": "2020-06-06 09:49:54",
    "latitude": 55.71294467,
    "longitude": 37.66542005,
    "altitude": 429.13,
    "speed": 55.5,
    "battery_voltage": null,
    "cabin_temperature": 18,
    "fuel_level": 32
}
{
    "device_id": "iv9a94th6rztooxh5ur2",
    "datetime": "2020-06-07 15:00:10",
    "latitude": 55.70985913,
    "longitude": 37.62141918,
    "altitude": 417.0,
    "speed": 15.7,
    "battery_voltage": 10.3,
    "cabin_temperature": 17,
    "fuel_level": null
}
```

{% endcut %}

## Настройте кластер-приемник {#configure-target}

{% note tip %}

Вы можете поставлять данные в кластер {{ mes-name }} от имени пользователя `admin`, имеющего роль `superuser`, но безопаснее для каждой задачи создавать отдельных пользователей с ограниченными привилегиями. Подробнее см. в разделе [{#T}](../../managed-elasticsearch/operations/cluster-users.md).

{% endnote %}

1. [Создайте роль]({{ links.es.docs }}/kibana/current/xpack-security.html#_roles_2) с привилегиями `create_index` и `write` для всех индексов (`*`).

1. [Создайте пользователя](../../managed-elasticsearch/operations/cluster-users.md) и назначьте ему эту роль.

## Подготовьте и активируйте трансфер {#prepare-transfer}

1. [Создайте эндпоинт](../../data-transfer/operations/endpoint/index.md#create) для [источника `{{ KF }}`](../../data-transfer/operations/endpoint/source/kafka.md):

    **{{ ui-key.yacloud.data-transfer.forms.section-endpoint }}**:

    * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaSource.connection.title }}**:

        * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaSourceConnection.connection_type.title }}** — `{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaConnectionType.managed.title }}`.

            * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.ManagedKafka.cluster_id.title }}** — выберите кластер-источник из списка.

            * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.ManagedKafka.auth.title }}** — **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.ManagedKafkaAuth.sasl.title }}**.

                * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.ManagedKafkaSASLAuth.user.title }}** — `mkf-user`.
                * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.ManagedKafkaSASLAuth.password.title }}** — укажите пароль пользователя.

        * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaTargetTopicSettings.topic.title }}** — `sensors`.

    * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaSource.advanced_settings.title }}** → **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaSourceAdvancedSettings.converter.title }}**:

        * **{{ ui-key.yc-data-transfer.data-transfer.console.form.kafka.console.form.kafka.KafkaSourceAdvancedSettings.converter.title }}** — `json`.
            * **{{ ui-key.yc-data-transfer.data-transfer.console.form.common.console.form.common.ConvertRecordOptions.data_schema.title }}** — `{{ ui-key.yc-data-transfer.data-transfer.console.form.common.console.form.common.DataSchema.json_fields.title }}`.

                Вставьте схему данных в формате JSON:

                {% cut "json" %}

                ```json
                [
                    {
                        "name": "device_id",
                        "type": "utf8",
                        "key": true
                    },
                    {
                        "name": "datetime",
                        "type": "utf8"
                    },
                    {
                        "name": "latitude",
                        "type": "double"
                    },
                    {
                        "name": "longitude",
                        "type": "double"
                    },
                    {
                        "name": "altitude",
                        "type": "double"
                    },
                    {
                        "name": "speed",
                        "type": "double"
                    },
                    {
                        "name": "battery_voltage",
                        "type": "double"
                    },
                    {
                        "name": "cabin_temperature",
                        "type": "uint16"
                    },
                    {
                        "name": "fuel_level",
                        "type": "uint16"
                    }
                ]
                ```

                {% endcut %}

1. [Создайте эндпоинт](../../data-transfer/operations/endpoint/index.md#create) для [приемника `{{ ES }}`](../../data-transfer/operations/endpoint/target/elasticsearch.md):

    **{{ ui-key.yacloud.data-transfer.forms.section-endpoint }}** → **{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchTarget.connection.title }}**:

    * **{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchConnection.connection_type.title }}** — `{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchConnectionType.mdb_cluster_id.title }}`

        * **{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchConnectionType.mdb_cluster_id.title }}** — выберите кластер-приемник из списка.

    * **{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchConnection.user.title }}** — укажите имя пользователя.

    * **{{ ui-key.yc-data-transfer.data-transfer.console.form.elasticsearch.console.form.elasticsearch.ElasticSearchConnection.password.title }}** — укажите пароль пользователя.

1. Создайте трансфер:

    {% list tabs %}

    - Вручную

        1. [Создайте трансфер](../../data-transfer/operations/transfer.md#create) типа **_{{ ui-key.yc-data-transfer.data-transfer.console.form.transfer.console.form.transfer.TransferType.increment.title }}_**, использующий созданные эндпоинты.
        1. [Активируйте трансфер](../../data-transfer/operations/transfer.md#activate) и дождитесь его перехода в статус **{{ ui-key.yacloud.data-transfer.label_connector-status-RUNNING }}**.

    - С помощью {{ TF }}

        1. Укажите в файле `data-transfer-mkf-mes.tf` переменные:

            * `source_endpoint_id` — значение идентификатора эндпоинта для источника;
            * `target_endpoint_id` — значение идентификатора эндпоинта для приемника;
            * `transfer_enabled` — значение `1` для создания трансфера.

        1. Проверьте корректность файлов конфигурации {{ TF }} с помощью команды:

            ```bash
            terraform validate
            ```

            Если в файлах конфигурации есть ошибки, {{ TF }} на них укажет.

        1. Создайте необходимую инфраструктуру:

            {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

        1. Трансфер активируется автоматически. Дождитесь его перехода в статус **{{ ui-key.yacloud.data-transfer.label_connector-status-RUNNING }}**.

    {% endlist %}

## Проверьте работоспособность трансфера {#verify-transfer}

Убедитесь, что в кластер {{ mes-name }} переносятся данные из топика кластера-источника {{ mkf-name }}:

1. Отправьте данные из файла `sample.json` в топик `sensors` {{ mkf-name }} с помощью утилит `jq` и `kafkacat`:

    ```bash
    jq -rc . sample.json | kafkacat -P \
       -b <FQDN хоста-брокера>:9091 \
       -t sensors \
       -k key \
       -X security.protocol=SASL_SSL \
       -X sasl.mechanisms=SCRAM-SHA-512 \
       -X sasl.username="mkf-user" \
       -X sasl.password="<пароль пользователя в кластере-источнике>" \
       -X ssl.ca.location={{ crt-local-dir }}{{ crt-local-file }} -Z
    ```

    Подробнее о настройке SSL-сертификата и работе с `kafkacat` см. в разделе [{#T}](../../managed-kafka/operations/connect.md).

1. Проверьте, что индекс `sensors` кластера {{ mes-name }} содержит отправленные данные:

    {% list tabs %}

    - Bash

        Выполните команду:

        ```bash
        curl \
            --user <имя пользователя в кластере-приемнике>:<пароль пользователя в кластере-приемнике> \
            --cacert ~/.elasticsearch/root.crt \
            --header 'Content-Type: application/json' \
            --request GET 'https://c-<идентификатор кластера {{ ES }}>.rw.{{ dns-zone }}:{{ port-mes }}/sensors/_search?pretty'
        ```

    - Kibana

        1. [Подключитесь](../../managed-elasticsearch/operations/cluster-connect.md#connection-string) к кластеру-приемнику с помощью Kibana.
        1. Откройте панель управления, нажав на значок ![os-dashboards-sandwich](../../_assets/os-dashboards-sandwich.svg).
        1. В разделе **Analytics** выберите **Discover**.
        1. В поле **Index pattern** выберите индекс `sensors`.

    {% endlist %}

## Удалите созданные ресурсы {#clear-out}

{% note info %}

Перед тем как удалить созданные ресурсы, [деактивируйте трансфер](../../data-transfer/operations/transfer.md#deactivate).

{% endnote %}

Некоторые ресурсы платные. Чтобы за них не списывалась плата, удалите ресурсы, которые вы больше не будете использовать:

1. [Удалите трансфер](../../data-transfer/operations/transfer.md#delete).
1. [Удалите эндпоинты](../../data-transfer/operations/endpoint/index.md#delete) для источника и приемника.

Остальные ресурсы удалите в зависимости от способа их создания:

{% list tabs %}

- Вручную

    * [Удалите кластер {{ mkf-name }}](../../managed-kafka/operations/cluster-delete.md).
    * [Удалите кластер {{ mes-name }}](../../managed-elasticsearch/operations/cluster-delete.md).

- С помощью {{ TF }}

    1. В терминале перейдите в директорию с планом инфраструктуры.
    1. Удалите конфигурационный файл `data-transfer-mkf-mes.tf`.
    1. Проверьте корректность файлов конфигурации {{ TF }} с помощью команды:

        ```bash
        terraform validate
        ```

        Если в файлах конфигурации есть ошибки, {{ TF }} на них укажет.

    1. Подтвердите изменение ресурсов.

        {% include [terraform-apply](../../_includes/mdb/terraform/apply.md) %}

        Все ресурсы, которые были описаны в конфигурационном файле `data-transfer-mkf-mes.tf`, будут удалены.

{% endlist %}
