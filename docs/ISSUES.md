## Docker-in-Docker не працював коректно

При спробі запустити Docker daemon і контейнери всередині ubu1/ubu2 виникали помилки 
(`unshare: operation not permitted`, daemon не стартував).

Причина: для повноцінної роботи Docker-in-Docker контейнеру потрібні додаткові привілеї 
(зокрема `--privileged` або capability `SYS_ADMIN`), яких за замовчуванням немає.

Рішення:
- Зовнішні контейнери (ubu1, ubu2) запускалися з прапором `--privileged`
- У деяких випадках використовував бінарники (Loki, Promtail) замість Docker-образів

## Promtail: несумісність з Docker API

При використанні `docker_sd_configs` Promtail видавав помилку: client version 1.42 is too old. Minimum supported API version is 1.44, please upgrade your client

Через це він не міг автоматично знаходити контейнери і збирати з них логи. 
**Причина:**  
Promtail версії 2.9.0 зібраний зі старою бібліотекою Docker client (підтримує API до 1.42), тоді як Docker daemon на ubu1 вимагав мінімум API 1.44.

**Рішення:**  
Відмовився від service discovery через Docker socket і перейшов на пряме читання json-логів контейнерів:

```yaml
scrape_configs:
  - job_name: nginx-modsec
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-modsec
          host: ubu1
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - docker: {}
