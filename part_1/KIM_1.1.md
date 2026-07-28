# КИМ 1.1. Введение, реляционная алгебра, создание и управление схемой реляционной БД (DDL, НФ, типы данных и т.д.)

Тип оценочного средства (предмет контроля): защита отчета

Перечень контролируемых тем (модулей): Модуль 1

## Цель
Формирование у обучающегося знаний по работе с реляционной схемой базы данных (БД), к котором относятся:
- операции управления схемой (создание, установка связей между отношениями, удаление)
- нормализация отношений (до нормальной формы Бойса-Кодда)
- наполнение данными
- манипулирование данными

### Задачи

1. Сформировать схему реляционной БД учебного центра, который описан с использованием следующего набора сущностей и атрибутов:

   - **Преподаватель** (персональные данные включая ФИО и номер(а) документа(ов), специальность, квалификация, стаж, предыдущие места работы, рейтинг, зарплата);
   - **Обучающийся** (персональные данные включая ФИО и номер(а) документа(ов), рейтинг, совершеннолетие, законные представители);
   - **Курсы** (название, аннотация, предшествующие курсы, длительность по времени, стоимость, специальное оборудование / программное обеспечение, преподаватели, обучающиеся);
   - **Журнал оценок** (преподаватель, обучающийся, курс, промежуточные оценки на курсе включая отдельные тесты, сданные задания, итоговая оценка);
   - **Расписание занятий** (преподаватель, курс, кабинет, обучающиеся, время начала, время окончания);
   - **Кабинеты** (тип кабинета, номер, специальное оборудование / программное обеспечение).
2. Выполнить нормализацию сформированной схемы реляционной БД до нормальной формы Бойса-Кодда включительно. Привести в отчете модифицированную схему реляционной БД с обоснованием принятых изменений в табличном формате:

   1. Нормальная форма.
   2. Отношение / отношения до нормализации к конкретной нормальной форме.
   3. Отношение / отношения после нормализации к конкретной нормальной форме.
3. Наполнить разработанную схему БД синтетическими осмысленными данными.
4. Разработать 5 сложных запросов, предназначенных для получения комплексной информации в сущностях (например, среднее количество студентов на всех курсах, максимальное количество занятий в аудиториях с задаваемой маской номера и так далее).

## Пример на тему "Автосервис"

### Скрипт создания
```
create table clients (
   id serial primary key,
   first_name varchar(50) not null,
   last_name varchar(50) not null
);

create table employees (
   id serial primary key,
   first_name varchar(50) not null,
   last_name varchar(50) not null,
   salary int not null check (salary > 0)
);

create table services (
   id serial primary key,
   title varchar(100) not null,
   price int not null check (price > 0)
);

create table cars (
   id serial primary key,
   client_id int references clients(id) on delete cascade not null,
   number varchar(9) not null unique
);

create table orders (
   id serial primary key,
   car_id int references cars(id) on delete cascade not null,
   employee_id int references employees(id) on delete set null,
   created_at timestamp default current_timestamp not null,
   status varchar(20) default 'ожидание' check (
      status in ('ожидание', 'в работе', 'выполнен', 'отменен')
   ) not null
);

create table orders_to_services (
   order_id int references orders(id) on delete cascade not null,
   service_id int references services(id) on delete restrict not null
);

create unique index idx_orderid_to_serviceid on orders_to_services(order_id, service_id);

create table reviews (
   id serial primary key,
   client_id int references clients(id) on delete cascade not null,
   text varchar(1000),
   rating int check (rating between 1 and 5) not null,
   created_at timestamp default current_timestamp not null
);

create table suppliers (
   id serial primary key,
   title varchar(100) not null
);

create table car_parts (
   id serial primary key,
   title varchar(100) not null,
   quantity int check (quantity >= 0) not null
);

create table car_parts_orders (
   id serial primary key,
   car_part_id int references car_parts(id) on delete restrict not null,
   supplier_id int references suppliers(id) on delete restrict not null,
   quantity int check (quantity >= 0) not null
);

create table storage_transactions (
   id serial primary key,
   car_part_id int references car_parts(id) on delete cascade not null,
   quantity int check (quantity > 0) not null,
   type varchar(11) check (type in ('поступление', 'списание')) not null
);

create table payments (
   id serial primary key,
   sum int check (sum > 0) not null,
   description varchar(100),
   type varchar(11) check (type in ('пополнение', 'списание')) not null
);

create table vacation_schedules (
   id serial primary key,
   employee_id int references employees (id) on delete cascade not null,
   start_date date not null,
   end_date date not null
);

create table incidents (
   id serial primary key,
   description varchar(1000) not null
);

create table promotions (
   id serial primary key,
   title varchar(50) not null,
   description varchar(1000) not null,
   start_date date not null,
   end_date date not null
);
```

## Критерии оценки

- 5 баллов выставляется студенту, если он своевременно выполнил все задачи, предусмотренные в лабораторной работе, подготовил отчет в соответствии с требованиями преподавателя и в процессе защиты продемонстрировал наличие теоретических знаний в объеме содержания учебной дисциплины, относящейся к лабораторной работе. Сумел ответить на дополнительные вопросы, связанные не только с процессом выполнения лабораторной работы, но и с пониманием совершенных действий и решенных задач;
- 4 балла выставляется студенту, если он выполнил от 80% задач, предусмотренных в лабораторной работе, подготовил отчет в соответствии с требованиями преподавателя и в процессе защиты продемонстрировал наличие теоретических знаний в объеме содержания учебной дисциплины, относящейся к лабораторной работе. Сумел ответить на вопросы, связанные с процессом выполнения лабораторной работы;
- 3 балла выставляется студенту, если он более чем на 50% выполнил поставленные в лабораторной работе задачи, способен ответить на вопросы, касающиеся теоретической составляющей в объеме содержания учебной дисциплины, относящейся к лабораторной работе;
- 0 баллов выставляется студенту, если он не выполнил поставленные в лабораторной работе задачи, не способен ответить на вопросы, касающиеся теоретической составляющей в объеме содержания учебной дисциплины, относящейся к лабораторной работе.
