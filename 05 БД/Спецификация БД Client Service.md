# Спецификация БД — Client Service

![Client_Service_ERD](https://raw.githubusercontent.com/nckmlshv/Spirit.Pro/main/docs/Client_Service_ERD.svg)

## clients

Ядро карточки клиента. Хранит персональные данные и текущий статус членства.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор клиента |
| client_number | VARCHAR(16) | Unique, Not Null | Человекочитаемый номер клиента для ресепшен |
| last_name | VARCHAR(100) | Not Null | Фамилия |
| first_name | VARCHAR(100) | Not Null | Имя |
| middle_name | VARCHAR(100) | Null | Отчество, может отсутствовать |
| birth_date | DATE | Not Null | Дата рождения. Неизменяемый реквизит, используется при поиске дублей |
| sex | ENUM | Not Null, Check(sex in ('MALE', 'FEMALE')) | Пол клиента: мужской, женский |
| phone | VARCHAR(20) | Unique, Not Null | Номер телефона в формате E.164, уникален в системе |
| email | VARCHAR(255) | Null | Электронная почта |
| address | VARCHAR(500) | Null | Адрес фактического проживания |
| status | ENUM | Not Null, Default 'POTENTIAL', Check(status in ('POTENTIAL', 'MEMBER', 'FORMER')) | Статус членства: ПЧК, ЧК, БЧК |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время создания карточки |
| created_by | UUID | Not Null | Субъект, создавший карточку |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |

---

## identity_documents

Документы, удостоверяющие личность клиента. У клиента один документ в статусе ACTIVE, прежние переводятся в EXPIRED.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор документа |
| client_id | UUID | FK → clients.id, Not Null | Клиент, которому принадлежит документ |
| document_type | ENUM | Not Null, Check(document_type in ('PASSPORT_RF', 'PASSPORT_INTERNATIONAL', 'FOREIGN_PASSPORT', 'BIRTH_CERTIFICATE', 'RESIDENCE_PERMIT', 'TEMPORARY_RESIDENCE', 'TEMPORARY_ID', 'REFUGEE_CERTIFICATE', 'MILITARY_TICKET', 'MILITARY_ID')) | Тип документа: паспорт гражданина РФ, загранпаспорт гражданина РФ, паспорт иностранного гражданина, свидетельство о рождении, вид на жительство иностранного гражданина, разрешение на временное проживание, временное удостоверение личности, удостоверение беженца, военный билет, удостоверение военнослужащего |
| issuing_country | CHAR(2) | Not Null | Код страны выдачи по ISO 3166-1 |
| series | VARCHAR(20) | Null | Серия документа, может отсутствовать |
| number | VARCHAR(30) | Not Null | Номер документа |
| issue_date | DATE | Not Null | Дата выдачи документа |
| expiry_date | DATE | Null | Срок действия. NULL для бессрочных документов |
| status | ENUM | Not Null, Default 'ACTIVE', Check(status in ('ACTIVE', 'EXPIRED')) | Статус документа: действующий, истёк срок действия |
| credentials | JSONB | Not Null | Реквизиты, специфичные для типа документа: орган выдачи, код подразделения, место рождения, гражданство |
| scans | JSONB | Null | Массив ссылок на сканы страниц документа в Document Service со штрихкодами |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время внесения документа |
| created_by | UUID | Not Null | Субъект, внёсший документ |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |

**Ограничения:** уникальный составной ключ `(document_type, issuing_country, series, number)` — двух карточек с одним номером ДУЛ быть не может.

---

## client_consents

Согласия клиента по 152-ФЗ. Отсутствие действующего согласия на обработку персональных данных блокирует обслуживание.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор согласия |
| client_id | UUID | FK → clients.id, Not Null | Клиент, предоставивший согласие |
| consent_type | ENUM | Not Null, Check(consent_type in ('PERSONAL_DATA', 'MARKETING')) | Тип согласия: на обработку персональных данных, на получение рекламных материалов |
| status | ENUM | Not Null, Check(status in ('GRANTED', 'REFUSED', 'REVOKED', 'EXPIRED')) | Статус согласия: предоставлено, клиент отказался, отозвано клиентом, истёк срок действия |
| granted_at | TIMESTAMPTZ | Null | Дата и время предоставления согласия |
| expires_at | TIMESTAMPTZ | Null | Срок действия согласия |
| withdrawn_at | TIMESTAMPTZ | Null | Дата и время отзыва согласия |
| withdrawal_reason | TEXT | Null | Причина отзыва согласия |
| document_id | UUID | Null | Скан подписанного согласия в Document Service |
| barcode | VARCHAR(64) | Unique, Null | Штрихкод печатной формы для автопривязки скана |
| granted_by_representative_id | UUID | FK → representatives.id, Null | Законный представитель, давший согласие за несовершеннолетнего |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время создания записи |
| created_by | UUID | Not Null | Субъект, оформивший согласие |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |

---

## representatives

Связь несовершеннолетнего клиента с законным представителем. Даёт представителю делегированный доступ к профилю подопечного.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор связи |
| client_id | UUID | FK → clients.id, Not Null | Несовершеннолетний клиент |
| representative_id | UUID | FK → clients.id, Not Null | Карточка законного представителя |
| relation_type | ENUM | Not Null, Check(relation_type in ('MOTHER', 'FATHER', 'STEPMOTHER', 'STEPFATHER', 'GRANDMOTHER', 'GRANDFATHER', 'AUNT', 'UNCLE', 'SISTER', 'BROTHER', 'OTHER_RELATIVE', 'NON_RELATIVE')) | Степень родства: мать, отец, мачеха, отчим, бабушка, дедушка, тётя, дядя, сестра, брат, иной родственник, не является родственником |
| status | ENUM | Not Null, Default 'ACTIVE', Check(status in ('ACTIVE', 'REVOKED', 'EXPIRED')) | Статус полномочий представителя: действительны, прекращены, истёк срок |
| document_type | ENUM | Null, Check(document_type in ('BIRTH_CERTIFICATE', 'PASSPORT_RECORD', 'ADOPTION_CERTIFICATE', 'GUARDIANSHIP_ACT', 'FOSTER_FAMILY_AGREEMENT', 'PATRONAGE_AGREEMENT', 'COURT_DECISION', 'POWER_OF_ATTORNEY', 'FOREIGN_DOCUMENT')) | Тип документа, подтверждающего родство или полномочия: свидетельство о рождении, отметка о ребёнке в паспорте родителя, свидетельство об усыновлении, акт органа опеки о назначении опекуна или попечителя, договор о приёмной семье, договор о патронатном воспитании, решение суда, нотариальная доверенность, документ иностранного государства |
| document_id | UUID | Null | Скан подтверждающего документа в Document Service |
| document_details | JSONB | Null | Реквизиты подтверждающего документа: номер, дата, орган выдачи |
| authority_expires_at | DATE | Null | Срок действия полномочий, если ограничен |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время создания связи |
| created_by | UUID | Not Null | Субъект, оформивший связь |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |

**Ограничения:** уникальный составной ключ `(client_id, representative_id)` — повторная связь между теми же клиентами невозможна.

---

## corporate_clients

Участие клиента в корпоративной программе. Оплата абонемента производится компанией-работодателем.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор участия |
| client_id | UUID | FK → clients.id, Not Null | Клиент — участник корпоративной программы |
| organization_id | UUID | Not Null | Компания-плательщик из справочника организаций |
| employee_number | VARCHAR(50) | Null | Табельный номер сотрудника в компании |
| status | ENUM | Not Null, Default 'ACTIVE', Check(status in ('ACTIVE', 'SUSPENDED', 'TERMINATED')) | Статус участия в программе: активно, приостановлено, прекращено |
| enrolled_at | DATE | Not Null | Дата включения в корпоративную программу |
| terminated_at | DATE | Null | Дата исключения из программы |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время создания записи |
| created_by | UUID | Not Null | Субъект, оформивший участие |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |

**Ограничения:** уникальный составной ключ `(client_id, organization_id)`.

---

## client_comments

Внутренние комментарии сотрудников к карточке клиента.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | UUID | PK, Not Null | Уникальный идентификатор комментария |
| client_id | UUID | FK → clients.id, Not Null | Клиент, к карточке которого оставлен комментарий |
| body | TEXT | Not Null | Текст комментария |
| club_id | UUID | Not Null | Клуб, в котором оставлен комментарий |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время создания комментария |
| created_by | UUID | Not Null | Субъект, оставивший комментарий |
| updated_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время последнего изменения |
| updated_by | UUID | Null | Субъект, внёсший последнее изменение |
| deleted_at | TIMESTAMPTZ | Null | Дата и время удаления комментария |
| deleted_by | UUID | Null | Субъект, удаливший комментарий |

---

## client_audit_log

Журнал операций с данными клиента. Запись выполняется в одной транзакции с изменением, что исключает потерю событий.

| Имя атрибута | Тип данных | Constraint | Описание |
|---|---|---|---|
| id | BIGINT | PK, Not Null, Auto Increment | Уникальный идентификатор записи журнала |
| client_id | UUID | FK → clients.id, Not Null | Клиент, к данным которого относится операция |
| entity_name | VARCHAR(64) | Not Null | Таблица, в которой произошло изменение |
| entity_id | UUID | Not Null | Идентификатор изменённой записи |
| action | ENUM | Not Null, Check(action in ('CREATE', 'UPDATE', 'DELETE')) | Тип выполненной операции: создание записи, изменение записи, удаление записи |
| old_data | JSONB | Null | Изменённые поля с прежними значениями |
| new_data | JSONB | Null | Изменённые поля с новыми значениями |
| created_at | TIMESTAMPTZ | Not Null, Default now() | Дата и время выполнения операции |
| created_by | UUID | Null | Субъект, выполнивший операцию |
| club_id | UUID | Null | Клуб, из которого выполнена операция |
| ip_address | INET | Null | IP-адрес источника запроса |
| user_agent | VARCHAR(500) | Null | Клиентское приложение, из которого выполнена операция |
| request_id | UUID | Null | Сквозной идентификатор запроса для трассировки |
