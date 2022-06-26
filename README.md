# Домашнее задание к занятию "08.02 Работа с Playbook"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
3. Подготовьте хосты в соотвтествии с группами из предподготовленного playbook. 
4. Скачайте дистрибутив [java](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html) и положите его в директорию `playbook/files/`. 

## Основная часть
1. Приготовьте свой собственный inventory файл `prod.yml`.

```
---
elasticsearch:
  hosts:
    localhost:
      ansible_connection: local
kibana:
  hosts:
    localhost:
      ansible_connection: local
```

2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.

```
- name: Install kibana
  hosts: kibana
  tasks:
    - name: Upload tar.gz kibana from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/kibana-{{ elastic_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: true
        validate_certs: false
      register: get_kibana
      until: get_kibana is succeeded
      tags: kibana
    - name: Create directrory for kibana
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}"
      tags: kibana
    - name: Extract Kibana in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
      tags:
        - kibana
    - name: Set environment Kibana
      become: true
      template:
        src: templates/kib.sh.j2
        dest: /etc/profile.d/kib.sh
      tags: kibana
```
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.

```
vagrant@vagrant:/vagrant/8.2$ ansible-lint site.yml -v
Examining site.yml of type playbook
```

6. Попробуйте запустить playbook на этом окружении с флагом `--check`.

```
vagrant@vagrant:/vagrant/8.2$ ansible-playbook -i inventory/prod.yml site.yml --check

PLAY [Install Java] ****************************************************************************************************************************************
TASK [Gathering Facts] *************************************************************************************************************************************ok: [localhost]

TASK [Set facts for Java 11 vars] **************************************************************************************************************************ok: [localhost]

TASK [Upload .tar.gz file containing binaries from local storage] ******************************************************************************************changed: [localhost]

TASK [Ensure installation dir exists] **********************************************************************************************************************changed: [localhost]

TASK [Extract java in the installation directory] **********************************************************************************************************An exception occurred during task execution. To see the full traceback, use -vvv. The error was: NoneType: None
fatal: [localhost]: FAILED! => {"changed": false, "msg": "dest '/opt/jdk/11.0.15.1' must be an existing dir"}

PLAY RECAP *************************************************************************************************************************************************localhost                  : ok=4    changed=2    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.

Это третий запуск плейбука. 

```
vagrant@vagrant:/vagrant/8.2$ ansible-playbook -i inventory/prod.yml site.yml --diff

PLAY [Install Java] ****************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [localhost]

TASK [Set facts for Java 11 vars] **************************************************************************************************************************
ok: [localhost]

TASK [Upload .tar.gz file containing binaries from local storage] ******************************************************************************************
ok: [localhost]

TASK [Ensure installation dir exists] **********************************************************************************************************************
ok: [localhost]

TASK [Extract java in the installation directory] **********************************************************************************************************
skipping: [localhost]

TASK [Export environment variables] ************************************************************************************************************************
ok: [localhost]

PLAY [Install Elasticsearch] *******************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [localhost]

TASK [Upload tar.gz Elasticsearch from remote URL] *********************************************************************************************************
ok: [localhost]

TASK [Create directrory for Elasticsearch] *****************************************************************************************************************
ok: [localhost]

TASK [Extract Elasticsearch in the installation directory] *************************************************************************************************
skipping: [localhost]

TASK [Set environment Elastic] *****************************************************************************************************************************
ok: [localhost]

PLAY [Install kibana] **************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [localhost]

TASK [Upload tar.gz kibana from remote URL] ****************************************************************************************************************
changed: [localhost]

TASK [Create directrory for kibana] ************************************************************************************************************************
ok: [localhost]

TASK [Extract Kibana in the installation directory] ********************************************************************************************************
changed: [localhost]

TASK [Set environment Kibana] ******************************************************************************************************************************
--- before
+++ after: /home/vagrant/.ansible/tmp/ansible-local-4305slw4z6hu/tmpcim43z_9/kib.sh.j2
@@ -0,0 +1,5 @@
+# Warning: This file is Ansible Managed, manual changes will be overwritten on next playbook run.
+#!/usr/bin/env bash
+
+export KIBANA_HOME=/opt/kibana/8.2.3
+export PATH=$PATH:$KIBANA_HOME/bin
\ No newline at end of file

changed: [localhost]

PLAY RECAP *************************************************************************************************************************************************
localhost                  : ok=14   changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.

```
PLAY RECAP *************************************************************************************************************************************************
localhost                  : ok=13   changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.


## GROUP_VARS

### ALL

- java_jdk_version - версия Java
- java_oracle_jdk_package: jdk-11.0.15.1_linux-aarch64_bin.tar.gz - название архива Java

### ELASTICSEARCH

- elastic_version - версия Elasticsearch
- elastic_home - домашний каталог Elasticsearch

### KIBANA

- kibana_version - версия Kibana
- kibana_home - - домашний каталог Kibana

## Play 

### Install Java

- Тег java
- Установка переменных
- Загрузка дистрибутива с локального репозитория
- Проверка, что директория установки существует
- Распаковка дистрибутива
- Экспорт переменных окружения по шаблону

### Install Elasticsearch

- Тег elastic
- Загрузка дистрибутива c удаленного репозитория
- Создание директории для Elasticsearch
- Распаковка дистрибутива в директорию инсталляции 
- Экспорт переменных окружения по шаблону

### Install kibana

- Тег kibana
- Загрузка дистрибутива c удаленного репозитория
- Создание директории для Kibana
- Распаковка дистрибутива в директорию инсталляции 
- Экспорт переменных окружения по шаблону




10. Готовый playbook выложите в свой репозиторий, в ответ предоставьте ссылку на него.
