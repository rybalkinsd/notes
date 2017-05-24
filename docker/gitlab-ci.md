# Continious delivery with docker and gitlab-ci
*\#docker \#gitlab \#gitlab-ci-multi-runner \#CI \#CD*

Недавно пришлось настраивать собственную систему CI на базе Gitlab. 
Эта заметка о том, что было сделано и зачем.

## Что будем делать?
Главная цель - полная автоматизация процесса тестирования и разворачивания в тестовом окружении после `git push`.
Еще хотелось бы иметь достаточно небольшой docker image для деплоя.
Весь процесс по-шагам должен выглядеть так так:

1. git push
1. сборка проекта
1. тестирование
1. разворачивание на тестовом стенде и деплой в бинарный репозиторий

## Ограничения
GitLab Community Edition 8.3.2 - главная проблема. 
Возможность обновить отсутствует.
В продакшене всё должно работать на достаточно толстом базовом image > 1Gb. 
Хочется обойтись общедоступными без потери функционала.

## build + test
Начну с простого - сборки и тестирования. 
Интеграция с gitlab-ci начинается с добавления в проект файла `.gitlab-ci.yml`

```yml
stages:
  - build
  - test

build:
  stage: build
  script:
    - mvn -U -DskipTests clean package

test:
  stage: test
  script:
    - mvn clean test verify
    - cat target/site/jacoco/index.html | egrep -o 'Total.*?[0-9]{1,3}%' | egrep -o '[0-9]{1,3}%' | head -1   
```

Раздел `stages` описывет, что последовательно будет делать c проектом gitlab. 
Сейчас у нас есть стадии `build` и `test`.
На стадии `test` в скриптах присутствует довольно страшная строчка с грепами. 
Она нужна, чтобы вытащить процент покрытия проекта тестами.
Кто-то может сказать, что gitlab позволяет делать `pages:` но не в моей версии, к сожалению. 
Так что выкручивался, как мог.

Хорошо, сборка и тестрирование есть. Переходим к деплойменту.

## deploy
На стадии `deploy` я ожидаю получить два результата:
- загруженный артефакт в бинарный репозиторий
- запущенный докер контейнер с текущей версией приложения

Давайте сначала разберемся с бинарным репозиторием'ом.

Для того, чтобы поддержать такой деплой, нужно три вещи:
1. Чтобы gitlab знал о существовании вашего репозитория
2. Добавить новый `stage` в `.gitlab-ci.yml`
3. Описать этот `stage`

```yaml
stages:
  ...
  - deploy
...
deploy-snapshot:
  stage: deploy
  script:
    - mvn -DskipTests deploy
  only:
    - master
```

Из конфига понятно, что деплоить я хочу только из ветки `master`. 
В этой фазе я уже могу пропустить тесты, которые благополучно прошли в предыдущей.

## docker
Осталось только научиться собирать образы совего приложения и деплоить их.
Из контекста понятно, что приложение собирается `maven`.
Код написан на java 8, значит нам еще понадобится подходящий `jre`.
Приложение должно быть доступно на тестовой машине - придется прокинуть какие-то порты.

Начнем с описания Dockerfile
```
FROM openjdk:8-jre-alpine

RUN mkdir -p /src/app
WORKDIR /src/app

COPY /path/to/target/app.jar /src/app/my-app.jar
EXPOSE <ports>

CMD ["java", "-jar", "my-app.jar"]
```

Для выполнения всех стадий CI gitlab использует runner'ы. 
Значит runner должен знать что такое jre, maven и docker.
Давайте создадим такой.
 
Сначала на тестовой машине необходимо установить docker. 
Мой докер - `Docker version 1.12.6, build 78d1802`.
Далее нужно поставить себе `gitlab-ci-multi-runner`, этот процесс подробно описан [тут](https://docs.gitlab.com/runner/install/linux-repository.html). 
Важно, чтобы API gitlab был совместим с API runner'а.
В этот момент стоит посмотреть [changelog](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/CHANGELOG.md).
Мне подошла версия предшествующая **v 9.0.0**.
В качестве executor'а в runner'е был выбран docker.

Основная конфигурация раннера находится в `/etc/gitlab-runner/config.toml` или внутри конкретного пользователя.
Что там можно прописывать, подробно описано [тут](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/configuration/advanced-configuration.md)

Распишу подробно только то, что попало в мой config.

```toml
[[runners]]
  name = "bears and vodka"
  url = ***
  token = ***
  executor = "docker"
  [runners.docker]
    image = "maven:3.5.0-jdk-8-onbuild"
    volumes = [
      "/cache", 
      "/.m2:/root/.m2", 
      "/var/run/docker.sock:/var/run/docker.sock", 
      "/usr/bin/docker:/usr/bin/docker"
    ]
  [runners.cache]
```

- image - базовый образ runner'а на нем мы будем делать всё то, что описано в `.gitlab-ci.yml`. maven:3.5.0-jdk-8-onbuild предоставляет нам java и mvn.
- url - ci url, можно взять в gitlab
- volume
  1. "/cache" хочется дать возможность кэшировать некоторые вещи для повторных запусков
  1. "/.m2:/root/.m2" - хочется иметь кастомный `settings.xml` и не скачивать зависимости каждый раз.
  1. Ну и наконец, нам нужно уметь собирать docker image, а для этого нужен docker.
     Самый простой способ затащить его в контейнер - последние две декларации в volume.
     
Ну и посмотрим на job deploy-docker в `.gitlab-ci.yml`

```yaml
deploy-docker:
  stage: deploy
  script:
    - mvn -DskipTests package
    - docker stop my-app || true
    - docker rm my-app || true
    - docker build -t my-app .
    - docker run -d -p 8080:7777 --name my-app my-app
  only:
    - master
```

Из этой декларации можно сделать следующие выводы:
1. Мы не только собираем docker image, но и сразу деплоим его в свою тестовую среду.
1. `build` и `run` выполняются в "одном и том же" инстансе докера.
1. Перед запуском контейнера происходит остановка и удаление предыдущего, если такой был.
1. Не делается push в приватное хранилище образов
1. Ожидается, что всегда master:head будет задеплоен

Последняя проблема с которой я столкнулся - dns. 
Корпоративные приложения часто хотят использовать собственные dns.
К счастью, настроить dns сразу для всех образов оказалось просто. 
Берем и прописываем в `/etc/docker/daemon.json

```json
{
    "dns": ["a.b.c.d", "w.x.y.z"]
}
```

## Немного итогов
Целью было быстро создать среду для тестирования, которая вписалась бы git-flow.
И вот что получилось:
1. Цепочка `git push` -> deployed container
1. Постоянно доступный для тестирования master:head
1. Минималистичный image < 150mb
1. Сеттинг полностью соответствующий окружению машины разработчика
