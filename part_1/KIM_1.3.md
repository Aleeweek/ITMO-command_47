# КИМ 1.3. Представления, индексы, процедуры, функции, триггеры и курсоры

Тип оценочного средства (предмет контроля): защита отчета

Перечень контролируемых тем (модулей): Модуль 1

## Цель

Формирование у обучающегося навыков создания и использования программных расширений СУБД (представлений, индексов, хранимых процедур, функций, триггеров и курсоров) для автоматизации обработки данных, обеспечения целостности и реализации бизнес-логики на стороне сервера БД.

### Задачи

Выполнение задач представленной лабораторной работы происходит с использованием схемы БД, разработанной в рамках выполнения [КИМ 1.1. Лабораторная работа](KIM_1.1.md).

1. Создать иерархию представлений (горизонтальных, вертикальных, объединённых) для предоставления различным ролям пользователей ограниченного и безопасного доступа к данным.
2. Обеспечить обновляемость представлений с использованием триггеров INSTEAD OF для сложных (объединённых и агрегированных) случаев.
3. Разработать набор индексов (включая составные и функциональные) и оценить их влияние на производительность операций выборки и модификации данных.
4. Написать хранимые процедуры и пользовательские функции (на PL/pgSQL или аналогичном процедурном расширении) для реализации повторяющихся бизнес-операций.
5. Реализовать триггеры для обеспечения недекларативных ограничений целостности и аудита действий пользователей (логирование изменений в служебную таблицу).
6. Продемонстрировать использование курсоров для построчной обработки данных в рамках серверной процедуры.

## Пример на тему "Автосервис"
### Триггер 1. Проверяет занятость механика перед назначением нового заказа
```
CREATE OR REPLACE FUNCTION CARSERVICE_5.CHECK_MECHANIC_AVAILABILITY() RETURNS TRIGGER AS $$
BEGIN
  IF (EXISTS (
    SELECT 1 FROM CARSERVICE_5.ORDERS WHERE EMPLOYEE_ID = NEW.EMPLOYEE_ID and STATUS = 'в работе'
  )) AND NEW.STATUS = 'в работе' THEN RAISE EXCEPTION 'Механик уже выполняет другой заказ';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE PLPGSQL;

CREATE TRIGGER TR_CHECK_MECHANIC_AVAILABILITY
BEFORE INSERT OR UPDATE ON CARSERVICE_5.ORDERS
FOR EACH ROW EXECUTE FUNCTION CARSERVICE_5.CHECK_MECHANIC_AVAILABILITY();
```

### Триггер 2. Контролирует даты создания заказа, запрещает создание заказов с прошедшей датой.
CREATE OR REPLACE FUNCTION CARSERVICE_5.CHECK_ORDER_DATE_NOT_IN_PAST()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.CREATED_AT < CURRENT_TIMESTAMP - INTERVAL '1 minute' THEN
        RAISE EXCEPTION 'Дата заказа не может быть в прошлом';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER TR_CHECK_ORDER_DATE_NOT_IN_PAST
BEFORE INSERT ON CARSERVICE_5.ORDERS
FOR EACH ROW
EXECUTE FUNCTION CARSERVICE_5.CHECK_ORDER_DATE_NOT_IN_PAST();





Триггер tr_check_order_date_not_in_past. Обновляет количество запчастей на складе при поступлении или списании, проверяет остатки на складе. 
CREATE OR REPLACE FUNCTION CARSERVICE_5.UPDATE_CAR_PARTS_QUANTITY()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.TYPE = 'поступление' THEN
        UPDATE CARSERVICE_5.CAR_PARTS
        SET QUANTITY = QUANTITY + NEW.QUANTITY
        WHERE ID = NEW.CAR_PART_ID;
    ELSIF NEW.TYPE = 'списание' THEN
        UPDATE CARSERVICE_5.CAR_PARTS
        SET QUANTITY = QUANTITY - NEW.QUANTITY
        WHERE ID = NEW.CAR_PART_ID;

        -- Проверка на отрицательный остаток
        IF (SELECT QUANTITY FROM CARSERVICE_5.CAR_PARTS WHERE ID = NEW.CAR_PART_ID) < 0 THEN
            RAISE EXCEPTION 'Недостаточно запчастей на складе для списания';
        END IF;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER TR_UPDATE_CAR_PARTS_QUANTITY
AFTER INSERT ON CARSERVICE_5.STORAGE_TRANSACTIONS
FOR EACH ROW
EXECUTE FUNCTION CARSERVICE_5.UPDATE_CAR_PARTS_QUANTITY();

 
Часть 3. Скалярные функции

Функция calculate_total_orders_price. Суммирует цены всех услуг по заказам со статусами "ожидание", "в работе" и "выполнен".
CREATE OR REPLACE FUNCTION CARSERVICE_5.CALCULATE_TOTAL_ORDERS_PRICE()
RETURNS BIGINT
AS $$
DECLARE RESULT INT;
BEGIN
  SELECT SUM(S.PRICE)
  INTO RESULT
  FROM CARSERVICE_5.ORDERS_TO_SERVICES OTS
  JOIN CARSERVICE_5.SERVICES S ON OTS.SERVICE_ID = S.ID
  JOIN CARSERVICE_5.ORDERS O ON OTS.ORDER_ID = O.ID
  WHERE STATUS IN ('ожидание', 'в работе', 'выполнен');

  RETURN RESULT;
END;
$$ LANGUAGE PLPGSQL;

Функция calculate_payments_by_type. Фильтрует платежи по типу и вычисляет их общую сумму.
CREATE OR REPLACE FUNCTION CARSERVICE_5.CALCULATE_PAYMENTS_BY_TYPE(PAYMENT_TYPE VARCHAR(11))
RETURNS BIGINT
AS $$
DECLARE RESULT INT;
BEGIN
  SELECT SUM(P.SUM)
  INTO RESULT
  FROM CARSERVICE_5.PAYMENTS P
  WHERE P.TYPE = PAYMENT_TYPE;
  RETURN RESULT;
END;
$$ LANGUAGE PLPGSQL;










Функция calculate_average_rating. Вычисляет среднее арифметическое всех оценок из отзывов.
CREATE OR REPLACE FUNCTION CARSERVICE_5.CALCULATE_AVERAGE_RATING()
RETURNS NUMERIC
AS $$
DECLARE RESULT NUMERIC;
BEGIN
  SELECT AVG(R.RATING)
  INTO RESULT
  FROM CARSERVICE_5.REVIEWS R;
  RETURN RESULT;
END;
$$ LANGUAGE PLPGSQL;

 
Часть 4. Представления

Представление orders_details. Детальная информация о заказах с итоговыми суммами.
CREATE OR REPLACE VIEW CARSERVICE_5.ORDERS_DETAILS AS
SELECT
    O.ID AS ORDER_ID,
    C.FIRST_NAME || ' ' || C.LAST_NAME AS CLIENT_NAME,
    CAR.NUMBER AS CAR_NUMBER,
    E.FIRST_NAME || ' ' || E.LAST_NAME AS EMPLOYEE_NAME,
    O.CREATED_AT,
    O.STATUS,
    SUM(S.PRICE) AS TOTAL_PRICE
FROM CARSERVICE_5.ORDERS_TO_SERVICES OTS
JOIN CARSERVICE_5.ORDERS O ON OTS.ORDER_ID = O.ID
JOIN CARSERVICE_5.SERVICES S ON OTS.SERVICE_ID = S.ID
JOIN CARSERVICE_5.CARS CAR ON O.CAR_ID = CAR.ID
JOIN CARSERVICE_5.CLIENTS C ON CAR.CLIENT_ID = C.ID
JOIN CARSERVICE_5.EMPLOYEES E ON O.EMPLOYEE_ID = E.ID
GROUP BY O.ID, CLIENT_NAME, CAR_NUMBER, EMPLOYEE_NAME, CREATED_AT, STATUS;

Представление client_ratings. Статистика по отзывам и рейтингам клиентов
CREATE OR REPLACE VIEW CARSERVICE_5.CLIENTS_RATINGS AS
SELECT
    C.ID AS CLIENT_ID,
    C.FIRST_NAME || ' ' || C.LAST_NAME AS CLIENT_NAME,
    COUNT(R.ID) AS TOTAL_REVIEWS,
    ROUND(AVG(R.RATING), 2) AS AVG_RATING
FROM CARSERVICE_5.CLIENTS C
JOIN CARSERVICE_5.REVIEWS R ON C.ID = R.CLIENT_ID
GROUP BY C.ID, C.FIRST_NAME, C.LAST_NAME;













Представление employees_availability. Отслеживание занятости сотрудников.
CREATE VIEW CARSERVICE_5.EMPLOYEES_AVAILABILITY AS
SELECT
    e.id AS employee_id,
    e.first_name,
    e.last_name,
    CASE 
        WHEN o.id IS NOT NULL AND o.status IN ('в работе', 'ожидание') THEN 'занят'
        ELSE 'свободен'
    END AS status,
    o.id AS current_order_id
FROM CARSERVICE_5.employees e
LEFT JOIN CARSERVICE_5.orders o ON e.id = o.employee_id AND o.status IN ('в работе', 'ожидание');

 
Часть 5. Индексы и ограничения

Индекс по (first_name, last_name) на таблице employees.  Гарантирует уникальность комбинации имени и фамилии сотрудника.
CREATE UNIQUE INDEX ON CARSERVICE_5.EMPLOYEES (FIRST_NAME, LAST_NAME);

Индекс idx_orderid_to_serviceid на таблице orders_to_services.  Обеспечивает уникальность связи заказ-услуга.
CREATE UNIQUE INDEX IDX_ORDERID_TO_SERVICEID ON CARSERVICE_5.ORDERS_TO_SERVICES (ORDER_ID, SERVICE_ID);

Индекс по (number) на таблице cars.  Гарантирует уникальность номеров автомобилей
CREATE UNIQUE INDEX ON CARSERVICE_5.CARS (NUMBER);

Ограничение chk_salary (таблица employees). Проверка корректности зарплаты сотрудников, то есть исключает нулевые или отрицательные значения зарплаты.
ALTER TABLE CARSERVICE_5.EMPLOYEES
ADD CONSTRAINT CHK_SALARY CHECK (SALARY > 0);

Ограничение chk_price (таблица services). Запрещает бесплатные или отрицательные цены на услуги.
ALTER TABLE CARSERVICE_5.SERVICES
ADD CONSTRAINT CHK_PRICE CHECK (PRICE > 0);

Ограничение chk_status (таблица orders). Обеспечивает корректность статусов заказов через ограничение допустимых статусов.
ALTER TABLE CARSERVICE_5.ORDERS
ADD CONSTRAINT CHK_STATUS CHECK (STATUS IN ('ожидание', 'в работе', 'выполнен', 'отменен'));



## Критерии оценки

- 5 баллов выставляется студенту, если он своевременно выполнил все задачи, предусмотренные в лабораторной работе, подготовил отчет в соответствии с требованиями преподавателя и в процессе защиты продемонстрировал наличие теоретических знаний в объеме содержания учебной дисциплины, относящейся к лабораторной работе. Сумел ответить на дополнительные вопросы, связанные не только с процессом выполнения лабораторной работы, но и с пониманием совершенных действий и решенных задач;
- 4 балла выставляется студенту, если он выполнил от 80% задач, предусмотренных в лабораторной работе, подготовил отчет в соответствии с требованиями преподавателя и в процессе защиты продемонстрировал наличие теоретических знаний в объеме содержания учебной дисциплины, относящейся к лабораторной работе. Сумел ответить на вопросы, связанные с процессом выполнения лабораторной работы;
- 3 балла выставляется студенту, если он более чем на 50% выполнил поставленные в лабораторной работе задачи, способен ответить на вопросы, касающиеся теоретической составляющей в объеме содержания учебной дисциплины, относящейся к лабораторной работе;
- 0 баллов выставляется студенту, если он не выполнил поставленные в лабораторной работе задачи, не способен ответить на вопросы, касающиеся теоретической составляющей в объеме содержания учебной дисциплины, относящейся к лабораторной работе.
