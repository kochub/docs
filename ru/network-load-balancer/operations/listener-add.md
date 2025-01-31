# Добавить обработчик к сетевому балансировщику

{% list tabs %}

- Консоль управления
  
  Чтобы добавить [обработчик](../concepts/listener.md) к сетевому балансировщику:
  
  1. В [консоли управления]({{ link-console-main }}) выберите каталог, где требуется добавить обработчик к балансировщику.
  1. В списке сервисов выберите **{{ network-load-balancer-name }}**.
  1. В строке балансировщика, к которому нужно добавить обработчик, нажмите значок ![image](../../_assets/horizontal-ellipsis.svg) и выберите **Добавить обработчик**.
  1. В открывшемся окне:
     
	 * Укажите порт, на котором обработчик будет принимать входящий трафик, из диапазона от 1 до 32767.
	 * Укажите целевой порт, на который балансировщик будет направлять трафик, из диапазона от 1 до 32767.
	 * Нажмите **Добавить**.
  
- CLI
  
  Если у вас еще нет интерфейса командной строки {{ yandex-cloud }}, [установите его](../../cli/quickstart.md#install).
  
  {% include [default-catalogue](../../_includes/default-catalogue.md) %}
  
  Чтобы добавить обработчик к сетевому балансировщику:
  
  1. Получите список балансировщиков:
  
     ```
     yc load-balancer network-load-balancer list
     +----------------------+--------------------+-------------+----------+----------------+------------------------+----------+
     |          ID          |        NAME        |  REGION ID  |   TYPE   | LISTENER COUNT | ATTACHED TARGET GROUPS |  STATUS  |
     +----------------------+--------------------+-------------+----------+----------------+------------------------+----------+
     | c58r8boim8qfkcqtuioj | test-load-balancer | ru-central1 | EXTERNAL |              0 |                        | INACTIVE |
     +----------------------+--------------------+-------------+----------+----------------+------------------------+----------+
  
     ```
  
  1. Добавьте обработчик, указав его имя, порт и версию IP-адреса:
  
     ```
     yc load-balancer network-load-balancer add-listener c580id04kvumgn7ssfh1 \
       --listener name=test-listener,port=80,external-ip-version=ipv4
     .....done
     id: c58r8boim8qfkcqtuioj
     folder_id: aoerb349v3h4bupphtaf
     created_at: "2019-04-01T09:29:25Z"
     name: test-load-balancer
     region_id: ru-central1
     status: INACTIVE
     type: EXTERNAL
     listeners:
     - name: test-listener
       address: <IP-адрес обработчика>
       port: "80"
       protocol: TCP
     ```
  
- API
  
  Добавить обработчик можно с помощью метода API [addListener](../api-ref/NetworkLoadBalancer/addListener.md).
  
{% endlist %}

## Примеры

### Добавление обработчика внутреннему сетевому балансировщику {#internal-listener}

{% list tabs %}

- CLI
  
  Выполните команду, указав имя обработчика, порт, идентификатор подсети и внутренний адрес из диапазона адресов подсети:
  
  ```
  yc load-balancer network-load-balancer add-listener b7rc2h753djb3a5dej1i \
    --listener name=test-listener,port=80,internal-subnet-id=e9b81t3kjmi0auoi0vpj,internal-address=10.10.0.14
  ```
  
{% endlist %}

