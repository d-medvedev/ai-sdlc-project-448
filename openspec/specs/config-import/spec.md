# Config Import Specification

## Purpose

Импорт и экспорт конфигурации симулятора: `src/app/config-file.ts` задаёт JSON-формат, проверяет файл целиком и возвращает либо готовую конфигурацию, либо список ошибок; `src/app/storage.ts` хранит конфигурацию в браузере в том же формате и откатывается к учебным данным, если сохранённое значение отсутствует или недействительно.

## Requirements

### Requirement: Файл конфигурации содержит целочисленную версию и четыре списка

`exportConfig` SHALL сериализовать конфигурацию как JSON-объект с целочисленным полем `version`, равным `CONFIG_VERSION`, и списками `users`, `groups`, `services`, `permissions`. Тот же формат SHALL использоваться при сохранении в браузере.

#### Scenario: Экспорт пишет целочисленную версию

- **WHEN** вызывается `exportConfig(seedConfig)`
- **THEN** в JSON есть `version`, равный `CONFIG_VERSION`, и он является целым числом

#### Scenario: Пустая конфигурация допустима

- **WHEN** экспортируется и импортируется конфигурация с четырьмя пустыми списками
- **THEN** результат равен `{ ok: true, config: { users: [], groups: [], services: [], permissions: [] } }`

### Requirement: Экспорт и импорт обратимы

`parseConfigFile` SHALL принимать результат `exportConfig` и возвращать конфигурацию, равную исходной.

#### Scenario: Круговая проверка учебных данных

- **WHEN** вызывается `parseConfigFile(exportConfig(seedConfig))`
- **THEN** результат равен `{ ok: true, config: seedConfig }`

### Requirement: Неизвестные поля игнорируются

`parseConfigFile` SHALL читать только известные поля и MUST NOT отклонять файл из-за неизвестных полей на корневом уровне или в записях.

#### Scenario: Лишние поля не мешают импорту

- **WHEN** в файле есть `exportedBy` на корневом уровне и `nickname` у пользователя
- **THEN** результат равен `{ ok: true, config: seedConfig }` без этих полей

### Requirement: Корень и версия проверяются до разбора списков

`parseConfigFile` SHALL отклонять текст, который не является JSON-объектом, и файл, у которого `version` отсутствует, не является целым числом или превышает `CONFIG_VERSION`.

#### Scenario: Текст не является JSON

- **WHEN** на вход подаётся `{oops`
- **THEN** результат `{ ok: false, errors: ["Файл не разбирается как JSON"] }`

#### Scenario: Корень не объект

- **WHEN** на вход подаётся `[]` или `null`
- **THEN** результат `{ ok: false, errors: ["Файл должен содержать объект конфигурации"] }`

#### Scenario: Версия отсутствует или нецелая

- **WHEN** поля `version` нет или оно равно строке `"1"`
- **THEN** результат `{ ok: false, errors: ["Нет поля version с целым числом"] }`

#### Scenario: Версия новее поддерживаемой

- **WHEN** `version` больше `CONFIG_VERSION`
- **THEN** результат содержит ошибку `Версия формата <version> новее поддерживаемой <CONFIG_VERSION>`

### Requirement: Списки и записи проверяются по форме

`parseConfigFile` SHALL требовать, чтобы каждый из четырёх разделов был списком объектов с непустыми строковыми полями; `user.groupIds` SHALL быть списком непустых строк; субъект разрешения SHALL быть объектом с типом `user` или `group`. Идентификаторы SHALL быть уникальны внутри своего списка.

#### Scenario: Отсутствующий список

- **WHEN** в файле нет `groups`
- **THEN** результат `{ ok: false, errors: ["Поле groups должно быть списком"] }`

#### Scenario: Неверное строковое поле

- **WHEN** `users[1].name` равно числу
- **THEN** результат `{ ok: false, errors: ["users[1]: поле name должно быть непустой строкой"] }`

#### Scenario: Повтор идентификатора

- **WHEN** двум сервисам задан один идентификатор `ledger`
- **THEN** результат содержит ошибку `services[1]: идентификатор «ledger» уже занят`

#### Scenario: Неверный тип субъекта

- **WHEN** `permissions[0].subject.type` равно `team`
- **THEN** результат `{ ok: false, errors: ["permissions[0]: тип субъекта должен быть user или group"] }`

#### Scenario: groupIds не список строк

- **WHEN** `users[0].groupIds` не является списком непустых строк
- **THEN** результат содержит ошибку `users[0]: поле groupIds должно быть списком строк`

### Requirement: Ссылочная целостность проверяется

После успешного разбора `parseConfigFile` SHALL проверять, что группы пользователей существуют, субъект разрешения ссылается на существующего пользователя или группу, а `serviceId` — на существующий сервис.

#### Scenario: Разрешение на несуществующего пользователя

- **WHEN** `permissions[0].subject.id` равно `nobody`
- **THEN** результат `{ ok: false, errors: ["permissions[0]: нет пользователя «nobody»"] }`

#### Scenario: Разрешение на несуществующую группу

- **WHEN** `permissions[1].subject.id` равно `ghosts`
- **THEN** результат `{ ok: false, errors: ["permissions[1]: нет группы «ghosts»"] }`

#### Scenario: Разрешение на несуществующий сервис

- **WHEN** `permissions[2].serviceId` равно `vault`
- **THEN** результат `{ ok: false, errors: ["permissions[2]: нет сервиса «vault»"] }`

#### Scenario: Пользователь в несуществующей группе

- **WHEN** `users[0].groupIds` содержит `ghosts`
- **THEN** результат `{ ok: false, errors: ["users[0]: нет группы «ghosts»"] }`

### Requirement: Возвращаются все ошибки, частичной конфигурации нет

`parseConfigFile` SHALL собирать все ошибки этапа разбора; если структурных ошибок нет, SHALL собирать все ошибки ссылочной целостности. При любой ошибке функция MUST NOT возвращать конфигурацию.

#### Scenario: Несколько ошибок ссылок

- **WHEN** в файле два разрешения ссылаются на несуществующие сервис и группу
- **THEN** результат содержит две ошибки, а не только первую

### Requirement: Сохранение использует формат экспорта

`saveConfig` SHALL записывать `exportConfig(config)` в `Storage` под ключом `access-policy-simulator:config`.

#### Scenario: Сохранённое значение совпадает с экспортом

- **WHEN** вызывается `saveConfig(storage, seedConfig)`
- **THEN** `storage.getItem(STORAGE_KEY)` равен `exportConfig(seedConfig)`

### Requirement: Загрузка возвращает сохранённую конфигурацию или учебные данные

`loadConfig` SHALL возвращать разобранную сохранённую конфигурацию, если она действительна, и MUST возвращать `seedConfig`, если значения нет, оно повреждено или нарушает ссылочную целостность.

#### Scenario: Сохранённая конфигурация возвращается

- **WHEN** в хранилище сохранена изменённая конфигурация
- **THEN** `loadConfig` возвращает её

#### Scenario: Ничего не сохранено

- **WHEN** в хранилище нет значения по ключу
- **THEN** `loadConfig` возвращает `seedConfig`

#### Scenario: Повреждённое значение

- **WHEN** в хранилище лежит `{oops`
- **THEN** `loadConfig` возвращает `seedConfig`

#### Scenario: Нарушена ссылочная целостность

- **WHEN** в сохранённой конфигурации нет сервисов, на которые ссылаются разрешения
- **THEN** `loadConfig` возвращает `seedConfig`

### Requirement: Ошибки хранилища не прерывают работу

`saveConfig` и `loadConfig` SHALL перехватывать исключения `Storage`; `loadConfig` MUST возвращать `seedConfig`, а `saveConfig` MUST NOT пробрасывать ошибку вызывающему коду.

#### Scenario: Хранилище недоступно при чтении

- **WHEN** `storage.getItem` выбрасывает исключение
- **THEN** `loadConfig` возвращает `seedConfig`

#### Scenario: Хранилище недоступно при записи

- **WHEN** `storage.setItem` выбрасывает исключение
- **THEN** `saveConfig` завершается без ошибки
