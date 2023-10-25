# Неделя 5: домашнее задание

## Перед тем как начать
- Как подготовить окружение [см. тут](./docs/01-prepare-environment.md)
- Как накатить миграции на базу данных [см. тут](./docs/02-data-migrations.md)
- **САМОЕ ВАЖНОЕ** - полное описание базы данных, схему и описание поле можно найти [тут](./docs/03-db-description.md)
- Воркшоп и примеры запросов [см. тут](https://gitlab.ozon.dev/cs/classroom-9/students/week-5/workshop-5/-/blob/master/README.md)

## Основные требования
- решением каждого задания является ОДИН SQL-запрос
- не допускается менять схему или сами данные, если этого явно не указано в задании
- поля в выборках должны иметь псевдоним (alias) указанный в задании
- решение необходимо привести в блоке каждой задачи ВМЕСТО комментария "ЗДЕСЬ ДОЛЖНО БЫТЬ РЕШЕНИЕ" (прямо в текущем readme.md файле)
- метки времени должны быть приведены в формат _dd.MM.yyyy HH:mm:ss_ (время в БД и выборках в UTC)

## Прочие пожелания
- всем будет удобно, если вы будете придерживаться единого стиля форматирования SQL-команд, как в [этом примере](./docs/04-sql-guidelines.md)

## Задание 1: 100 заданий с самым долгим временем выполнения
Время, затраченное на выполнение задания - это период времени, прошедший с момента перехода задания в статус "В работе" и до перехода в статус "Выполнено".
Нужно вывести 100 заданий с самым долгим временем выполнения. 
Полученный список заданий должен быть отсортирован от заданий с наибольшим временем выполнения к заданиям с наименьшим временем выполнения.

Замечания:
- Невыполненные задания (не дошедшие до статуса "Выполнено") не учитываются.
- Когда исполнитель берет задание в работу, оно переходит в статус "В работе" (InProgress) и находится там до завершения работы. После чего переходит в статус "Выполнено" (Done).
  В любой момент времени задание может быть безвозвратно отменено - в этом случае оно перейдет в статус "Отменено" (Canceled).
- Нет разницы выполняется задание или подзадание.
- Выборка должна включать задания за все время.

Выборка должна содержать следующий набор полей:
- номер задания (task_number)
- заголовок задания (task_title)
- название статуса задания (status_name)
- email автора задания (author_email)
- email текущего исполнителя (assignee_email)
- дата и время создания задания (created_at)
- дата и время первого перехода в статус В работе (in_progress_at)
- дата и время выполнения задания (completed_at)
- количество дней, часов, минут и секнуд, которые задание находилось в работе - в формате "dd HH:mm:ss" (work_duration)

### Решение
```sql
  select t.id as task_number
       , t.title as task_title
       , t.created_at as created_at
       , (select ts.name
          from task_statuses ts
          where ts.id = 4) as status_name
       , (select u.email
          from users u
          where u.id = tl.created_by_user_id) as author_email
       , (select u.email
          from users u
          where u.id = tl.assigned_to_user_id) as assignee_email
       , tl.at as in_progress_at
       , t.completed_at as completed_at
       , to_char(t.completed_at - tl.at, 'dd HH24:MI:SS') as work_duration
    from tasks t
    join task_logs tl on tl.task_id = t.id
   where tl.status = 3 /* InProgress */
     and t.completed_at is not null
     and t.id not in (select distinct tl.id
                        from task_logs tl
                       where tl.status = 5 /* Cancelled */)
   order by work_duration desc
   limit 100;
```

## Задание 2: Выбора для проверки вложенности
Задания могу быть простыми и составными. Составное задание содержит в себе дочерние - так получается иерархия заданий.
Глубина иерархии ограничено Н-уровнями, поэтому перед добавлением подзадачи к текущей задачи нужно понять, может ли пользователь добавить задачу уровнем ниже текущего или нет. Для этого нужно написать выборку для метода проверки перед добавлением подзадания, которая бы вернула уровень вложенности указанного задания и полный путь до него от родительского задания.

Замечания:
- ИД проверяемого задания передаем в sql как параметр _:parent_task_id_
- если задание _Е_ находится на 5м уровне, то путь должен быть "_//A/B/C/D/E_".

Выбора должна содержать:
- только 1 строку
- поле "Уровень задания" (level) - уровень указанного в параметре задания
- поле "Путь" (path)

### Решение
```sql
    with recursive cte as (select t.id
                                , t.parent_task_id
                                , 2 as level
                                , '/' || t.parent_task_id::text || '/' || t.id as path
                           from tasks t
                           where t.parent_task_id = :parent_task_id
                           union all
                          select t1.id
                                , t1.parent_task_id
                                , c.level + 1 as level
                                , c.path || '/' || t1.id::text as path
                            from tasks t1
                            join cte c on c.id = t1.parent_task_id)
    select c.id
         , c.level
         , c.path
      from cte c
     order by level desc
     limit 1;
```

## Задание 3 (за алмазик): Получить переписку по заданию в формате "вопрос-ответ"
По заданию могут быть оставлены комментарии от Автора и Исполнителя. Автор задания - это пользователь, создавший задание, Исполнитель - это актуальный пользователь, назначенный на задание.
У каждого комментария есть поле _author_user_id_ - это идентификатор пользователя, который оставил комментарий. Если этот идентикатор совпадает с идентификатором Автора задания, то сообщение должно отобразиться **в левой колонке**, следующее за ним сообщение-ответ Исполнителя (если _author_user_id_ равен ИД исполнителя) должно отобразиться **на той же строчке**, что и сообщение Автора, но **в правой колонке**. Считаем, что Автор задания задает Вопросы, а Исполнитель дает на них Ответы.
Выборка должны включать "беседы" по 5 самым новым зданям и быть отсортирвана по порядку отправки комментариев в рамках задания. 

Замечания:
- Актуальный исполнитель - это пользователь на момент выборки указан в поле assiged_to_user_id.
- Если вопроса или ответа нет, в соответствующем поле должен быть NULL.
- Считаем, что все сообщения были оставлены именно текущим исполнителем (без учета возможных переназначений).
- Если комментарий был оставлен не Автором и не Исполнителем, игнорируем его

Выборка должна содержать следующий набор полей:
- номер задания (task_number)
- email автора задания (author_email)
- email АКТУАЛЬНОГО исполнителя (assignee_email)
- вопрос (question)
- ответ (answer)
- метка времени, когда был задан вопрос (asked_at)
- метка времени, когда был дан ответ (answered_at)

<details>
  <summary>Пример</summary>

Переписка по заданию №1 между author@tt.ru и assgnee@tt.ru:
- 01.01.2023 08:00:00 (автор) "вопрос 1"
- 01.01.2023 09:00:00 (исполнитель) "ответ 1"
- 01.01.2023 09:15:00 (исполнитель) "ответ 2"
- 01.01.2023 09:30:00  (автор) "вопрос 2"

Ожидаемый результат выполнения SQL-запроса:

| task_number | author_email    | assignee_email | question  | answer  | asked_at             | answered_at          |
|-------------|-----------------|----------------|-----------|---------|----------------------|----------------------|
| 1           | author@tt.ru    | assgnee@tt.ru  | вопрос 1  | ответ 1 | 01.01.2023 08:00:00  | 01.01.2023 09:00:00  |
| 1           | author@tt.ru    | assgnee@tt.ru  | вопрос 1  | ответ 2 | 01.01.2023 08:00:00  | 01.01.2023 09:15:00  |
| 1           | author@tt.ru    | assgnee@tt.ru  | вопрос 2  |         | 01.01.2023 09:30:00  |                      |

</details>


### Решение
```sql
with newest_tasks as (select t.id
                           , t.created_by_user_id
                           , t.assigned_to_user_id
                      from tasks t
                      order by t.created_at desc
                      limit 5)
   , questions as (select tc.id
                        , tc.task_id
                        , tc.author_user_id
                        , tc.message
                        , tc.at
                        , case when lead(tc.at) over (partition by tc.task_id order by tc.at) is not null
                               then lead(tc.at) over (partition by tc.task_id order by tc.at)
                               else (timestamp '9999-01-01')
                           end as next_at
                     from tasks t
                     join task_comments tc on tc.task_id = t.id
                    where t.created_by_user_id = tc.author_user_id
                    order by tc.at)
   , answers as (select tc.id
                      , tc.task_id
                      , tc.author_user_id
                      , tc.message
                      , tc.at
                   from tasks t
                   join task_comments tc on tc.task_id = t.id
                  where t.assigned_to_user_id = tc.author_user_id
                  order by tc.at)
    select t.id as task_number
         , t.created_by_user_id as task_author
         , t.assigned_to_user_id as task_assignee
         , (select u.email
            from users u
            where u.id = t.created_by_user_id) as author_email
         , (select u.email
            from users u
            where u.id = t.assigned_to_user_id) as assignee_email
         , q.message as question
         , a.message as answer
         , q.at as asked_at
         , a.at as answered_at
      from newest_tasks t
 left join questions q on q.task_id = t.id
                      and q.author_user_id = t.created_by_user_id
 left join answers a on a.task_id = q.task_id
                    and a.at >= q.at
                    and a.at < q.next_at
     where q.author_user_id = t.created_by_user_id
     order by task_number, answered_at;
```
