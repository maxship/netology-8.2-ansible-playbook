### Файл плейбука c описанием параметров:


```yml
---
- name: Install Java # play для установки JDK
  hosts: all # будет выполняться для всех хостов
  tasks:
    - name: Set home dir for Java 11 
      set_fact:
        java_home: "/opt/jdk/{{ java_jdk_version }}" # задание пути к установочной директории java
      tags: java # все таски из этого плея помечены тегом "java"
    - name: Upload .tar.gz file containing binaries from local storage
      copy: # копирование файлов на удаленные хосты
        src: "{{ java_oracle_jdk_package }}" # локальный путь к копируемому файлу/директории
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz" # путь для копирования на удаленном сервере
        mode: 0644 # права на копируемый файл
      register: download_java_binaries # запись результата работы таска
      until: download_java_binaries is succeeded # повторение таски, пока не будет успешно выполнена распаковка
      tags: java
    - name: Ensure installation dir exists # 
      become: true # повышение прав
      file:
        state: directory # если директория не существует, она будет создана
        path: "{{ java_home }}" # путь к установочной директории
        mode: 0644
      tags: java
    - name: Extract java in the installation directory
      become: true
      unarchive: # разархивирование
        copy: false # если false, плагин ищет архив на управляемом хосте
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz" # путь к архиву на управляемом хосте
        dest: "{{ java_home }}" # абсолютный путь на удаленном хосте для распаковки
        extra_opts: [--strip-components=1] # дополнительный аргумент, в данном случае удаляет верхнюю директорию при распаковке
        creates: "{{ java_home }}/bin/java" # если указанный абсолютный путь уже существует, таск не будет выполняться
        mode: 0644
      tags:
        - java
    - name: Export environment variables
      become: true
      template: # данный шаблон предназначен для копирования на управляемый хост скрипта с переменными окружения
        src: jdk.sh.j2 # имя шаблона в папке templates на управляющем хосте.
        dest: /etc/profile.d/jdk.sh # путь на удаленном хосте, куда будет вставлен шаблон
        mode: 0644 
      tags: java

- name: Install Elasticsearch # play для установки elc
  hosts: elasticsearch # будет установлен на хостах группы "elasticsearch"
  tasks:
    - name: Upload tar.gz Elasticsearch from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz" 
        dest: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz" # абсолютный путь, куда будет скопирован файл
        mode: 0755 # права
        timeout: 60  # таймаут запроса
        force: true # если файл существует, он будет перезаписан
        validate_certs: false # проверка SSL сертификата
      register: get_elastic 
      until: get_elastic is succeeded
      tags: elastic
    - name: Create directrory for Elasticsearc
      become: true
      file:
        state: directory
        path: "{{ elastic_home }}"
        mode: 0644
      tags: elastic
    - name: Extract Elasticsearch in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch"
        mode: 0644
      tags:
        - elastic
    - name: Set environment Elastic
      become: true
      template: # данный шаблон предназначен для копирования на управляемый хост скрипта с переменными окружения
        src: templates/elk.sh.j2
        dest: /etc/profile.d/elk.sh
        mode: 0644
      tags: elastic

- name: Install Kibana
  hosts: kibana # будет установлен на хостах группы "kibana"
  tasks:
    - name: Upload tar.gz Kibana from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: true
        validate_certs: false
      register: get_kibana
      until: get_kibana is succeeded
      tags: kibana
    - name: Create directrory for Kibana
      become: true
      file:
        state: directory
        path: "{{ kibana_home }}"
        mode: 0644
      tags: kibana
    - name: Extract Kibana in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
        mode: 0644
      tags:
        - kibana
    - name: Set environment kibana
      become: true
      template:
        src: templates/kibana.sh.j2
        dest: /etc/profile.d/kibana.sh
        mode: 0644
      tags: kibana
      
```