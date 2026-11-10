# Модель базы данных ToDoList

PostgreSQL хранит пользователей Telegram, их списки и задачи. Модель соответствует классам `Core/Entities`; запросы соответствуют читающим методам `IToDoRepository`.

## Состав

| Файл | Назначение |
| --- | --- |
| `ToDoListDb.sql` | Таблицы, первичные и внешние ключи, ограничения и индексы |
| `ToDoListDb_Insert.sql` | Тестовые данные: 2 пользователя, 3 списка, 9 задач |
| `ToDoListDb_Select.sql` | Параметризованные запросы и примеры их выполнения |
| `Core/` | Сущности и интерфейс репозитория на C# |
| `tests/VerifyDatabase.sql` | Проверки связей, ограничений и операций удаления |

## Создание базы

Нужны PostgreSQL 16 или новее и доступ к утилитам `createdb` и `psql`. Выполните команды из папки решения. `postgres` - пользователь PostgreSQL; при запросе введите его пароль. Для сервера с другим портом замените `5432`.

```powershell
createdb -h localhost -p 5432 -U postgres ToDoList
psql -X -h localhost -p 5432 -U postgres -d ToDoList -v ON_ERROR_STOP=1 -f ToDoListDb.sql
psql -X -h localhost -p 5432 -U postgres -d ToDoList -v ON_ERROR_STOP=1 -f ToDoListDb_Insert.sql
psql -X -h localhost -p 5432 -U postgres -d ToDoList -v ON_ERROR_STOP=1 -f ToDoListDb_Select.sql
```

Применяйте первые два скрипта к новой пустой базе. Имена таблиц, столбцов, индексов и ограничений заключены в двойные кавычки, поэтому регистр сохраняется.

## Структура данных

| Таблица | Первичный ключ | Связи |
| --- | --- | --- |
| `ToDoUser` | `UserId`, UUID | Уникальный `TelegramUserId`, BIGINT |
| `ToDoList` | `Id`, UUID | `UserId` ссылается на `ToDoUser.UserId` |
| `ToDoItem` | `Id`, UUID | `UserId` ссылается на пользователя, nullable `ListId` - на список |

Навигационные свойства `User` и `List` представлены в SQL внешними ключами `UserId` и `ListId`. Составной внешний ключ дополнительно запрещает присваивать задаче список другого пользователя. Все внешние ключи имеют индексы; индекс `(ListId, UserId)` также обслуживает поиск по одному `ListId`.

При удалении списка его задачи сохраняются с `ListId = NULL`. Удаление пользователя удаляет принадлежащие ему списки и задачи. Пустые названия, повторяющиеся `TelegramUserId` и неизвестные состояния запрещены.

`State = 0` означает `Active`, `State = 1` - `Completed`. У выполненной задачи заполнено `StateChangedAt`; у повторно активированной задачи дата последней смены состояния может сохраняться. `Deadline` обязателен. Даты C# передаются в UTC и хранятся как `timestamp with time zone`. Отсутствующий Telegram username хранится пустой строкой.

## Запросы репозитория

В `ToDoListDb_Select.sql` для каждого метода объявлен одноимённый `PREPARE`. Параметры `$1`, `$2` имеют указанные типы; `EXECUTE` в конце файла показывает применение к тестовым данным. После выполнения подготовленные запросы освобождаются.

| Метод | Параметры | Результат |
| --- | --- | --- |
| `GetAllByUserId` | UUID пользователя | Все его задачи |
| `GetActiveByUserId` | UUID пользователя | Его активные задачи |
| `Get` | UUID задачи | Одна задача или пустой результат |
| `ExistsByName` | UUID пользователя, точное имя | Наличие имени среди всех его задач |
| `CountActive` | UUID пользователя | Число активных задач как INTEGER |
| `Find` | UUID пользователя | Кандидаты для переданного C#-предиката |
| `GetByUserIdAndList` | UUID пользователя, UUID списка или NULL | Все состояния в указанном списке |
| `GetCompletedByUserIdAndList` | UUID пользователя, UUID списка или NULL | Выполненные задачи в указанном списке |

Для `Find` приложение применяет `Func<ToDoItem, bool>` к полученным кандидатам. `NULL` в параметре списка выбирает только задачи без списка. Коллекции упорядочены по `CreatedAt`, затем по `Id`.

Методы `Add`, `Update` и `Delete` изменяют данные; в файл выборок входят восемь читающих методов интерфейса.

## Сборка и проверки

Для Core нужен .NET SDK 8.0.4xx.

```powershell
dotnet build ToDoList.sln --configuration Release
psql -X -h localhost -p 5432 -U postgres -d ToDoList -v ON_ERROR_STOP=1 -f tests/VerifyDatabase.sql
```

Проверки выполняются после заполнения базы. Изменения тестовой транзакции откатываются.

Задание: [«Модель базы данных», OTUS C# Basic](https://github.com/OTUS-NET/C-Sharp-Basic/blob/main/Homeworks/13%20%D0%9C%D0%BE%D0%B4%D0%B5%D0%BB%D1%8C%20%D0%B1%D0%B0%D0%B7%D1%8B%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85/Task.md). Сдача: [правила отправки домашнего задания](https://github.com/OTUS-NET/C-Sharp-Basic/blob/main/Homeworks/README.md).
