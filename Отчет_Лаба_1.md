ЛР 1. Loki + Zabbix + Grafana  
Отчет по этапам из методички (Nextcloud + логирование + мониторинг + визуализация).
### Команда: Абакунов Кирилл, Черепня Ярослав, Тимаков Егор

## Цель
- Подключить к тестовому сервису Nextcloud логирование через Promtail+Loki, мониторинг через Zabbix и визуализацию всего стека в Grafana.

## Подготовка окружения
- Создан `docker-compose.yml` с сервисами: `nextcloud`, `loki`, `promtail`, `grafana`, `postgres-zabbix`, `zabbix-server` и `zabbix-web-nginx-pgsql`, общими volume `nc-data` (логи Nextcloud) и `zabbix-db`.
- Создан конфиг `promtail_config.yml` для отправки логов Nextcloud в Loki:
  - `http_listen_port: 9080`, `grpc_listen_port: 0`.
  - `positions: /tmp/positions.yaml`.
  - `clients: http://loki:3100/loki/api/v1/push`.
  - `scrape_configs`: `job_name: system`, `labels.job: nextcloud_logs`, путь к логам `__path__: /opt/nc_data/*.log`.
- Запуск стенда: `docker compose up -d`. Проверка статуса: `docker compose ps`.

![telegram-cloud-photo-size-2-5359433391259979277-y](https://github.com/user-attachments/assets/534b80b4-3e71-47c4-91de-9d379d39c36b)


## Часть 1. Логирование
1) Инициализация Nextcloud  
   - Открыт веб-интерфейс по `http://localhost:8080`, создана учетная запись.  
   - Убедились, что файл `/var/www/html/data/nextcloud.log` появился и пополняется.
  ![telegram-cloud-photo-size-2-5359433391259979296-y](https://github.com/user-attachments/assets/17a5cb59-45a8-4141-b536-81415b0c44c0)
2) Проверка Promtail  
   - Просмотр логов `promtail`: строки `msg=Seeked /opt/nc_data/nextcloud.log` подтверждают, что лог-файл подхвачен.
<img width="1704" height="235" alt="image" src="https://github.com/user-attachments/assets/06f6e3f2-2014-4441-bcb6-b071a9c590ea" />


3) Проверка Loki  
   - Убедились, что сервис `loki:3100` доступен и принимает push от Promtail.
   

## Часть 2. Мониторинг (Zabbix)
1) Доступ к Zabbix  
   - Вход в веб-интерфейс `http://localhost:8082` (Admin | zabbix).
2) Импорт шаблона  
   - Создан `template.yml` из методички (HTTP_AGENT item `nextcloud.ping`, триггер на maintenance mode). Импортирован через Data collection → Templates → Import.
3) Разрешение имени Nextcloud  
   - В контейнере Nextcloud под пользователем `www-data` выполнено:  
     `php occ config:system:set trusted_domains 1 --value="nextcloud"`.
4) Создание хоста  
   - Data collection → Hosts → Create host. Адрес/имя: `nextcloud`, группа: Applications. Подключен шаблон `Test ping template`.
5) Проверка данных и триггера  
   - Monitoring → Latest data: статус `healthy`.  
   - Для проверки триггера включался maintenance mode:  
     `php occ maintenance:mode --on` / `--off`, после чего проблема в Monitoring → Problems отмечалась как решенная.
![telegram-cloud-photo-size-2-5359433391259979309-y](https://github.com/user-attachments/assets/91a81798-f0f6-434e-803d-dd058dc9491b)
![telegram-cloud-photo-size-2-5359433391259979310-y](https://github.com/user-attachments/assets/f27b4fe0-5fa5-4c43-9291-891c19dc0226)
![telegram-cloud-photo-size-2-5359433391259979311-y](https://github.com/user-attachments/assets/140c2c1c-e21a-49d9-ad42-6b4032e3dee0)
![telegram-cloud-photo-size-2-5359433391259979321-y](https://github.com/user-attachments/assets/c7355f21-bf2d-4dbf-8fa1-b90f2cd0b75d)
![telegram-cloud-photo-size-2-5359433391259979322-y](https://github.com/user-attachments/assets/1ae6bdaa-fd1d-4d42-bd7f-04ac282a6aa8)


## Часть 3. Визуализация (Grafana)
1) Установка плагина Zabbix  
   - `docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"`  
   - `docker restart grafana`
2) Активация плагина  
   - Grafana → Administration → Plugins → Zabbix → Enable.
3) Подключение датасурсов  
   - Loki: Connections → Data sources → Loki, URL `http://loki:3100`, Save & Test.  
   - Zabbix: Data source Zabbix, URL `http://zabbix-front:8080/api_jsonrpc.php`, креды Admin/zabbix, Save & Test.
![telegram-cloud-photo-size-2-5359433391259979324-y](https://github.com/user-attachments/assets/eda4d6e5-40e5-4672-a646-bab56954effb)
![telegram-cloud-photo-size-2-5359433391259979327-y](https://github.com/user-attachments/assets/ba429b97-53bd-4079-950d-8c9c1b61db8f)

4) Просмотр данных  
   - Explore: выбор `job` или `filename` из Loki для просмотра логов Nextcloud.
   ![telegram-cloud-photo-size-2-5359433391259979328-y](https://github.com/user-attachments/assets/9c7e5566-3421-4281-ac11-31478a8a3aa7)

   - Explore Zabbix: данные по item `nextcloud.ping`.
  ![telegram-cloud-photo-size-2-5359433391259979329-y](https://github.com/user-attachments/assets/49828ec1-ad36-47fd-a9d6-b05ca5c74c50)

5) Дашборды  
   Создан простой дашборд
    ![telegram-cloud-photo-size-2-5359433391259979343-y](https://github.com/user-attachments/assets/f08af280-5350-4f88-a78f-952cc55a1168)


## Задание и выполненные действия
- «Поиграться» с запросами в Explore (Loki/Zabbix) - выполнено.
- Два дашборда (Zabbix статус + Loki таблица) - собраны.

## Ответы на вопросы
- Чем SLO отличается от SLA?  
  SLO - внутреннее целевое значение надежности/качества (например, 99.9% аптайм). SLA - юридически или договором закрепленные обязательства перед клиентом с последствиями при нарушении (штрафы/кредиты); SLA обычно опирается на одно или несколько SLO.
- Чем отличается инкрементальный бэкап от дифференциального?  
  Инкрементальный содержит изменения только со времени последнего бэкапа любого типа; цепочка восстанавливается из полного + всех последующих инкрементов. Дифференциальный содержит изменения с момента последнего полного бэкапа; для восстановления нужен полный + последний дифференциальный.
- В чем разница между мониторингом и observability?  
  Мониторинг - сбор и отображение заранее определенных метрик/сигналов для отслеживания известного состояния. Observability - способность системы давать достаточно телеметрии (метрики, логи, трассировки), чтобы отвечать на новые/неизвестные вопросы о ее поведении без добавления нового кода.

## Итог
- Развернут стенд Nextcloud + Loki/Promtail + Zabbix + Grafana по методичке, логирование и мониторинг работают, источники подключены в Grafana, базовые дашборды созданы.

## Выводы (чему научился)
- Поднимать связку Nextcloud + Loki/Promtail + Zabbix + Grafana через docker compose.
- Настраивать сбор логов Promtail и отправку в Loki, проверять доступность логов через Explore.
- Импортировать и применять шаблоны Zabbix, заводить хосты и проверять триггеры (maintenance mode).
- Подключать Zabbix и Loki как источники данных в Grafana и строить простые дашборды (статус и таблица логов).
- Понимать разницу между SLO и SLA, типами бэкапов, а также между мониторингом и observability.

