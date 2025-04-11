# Лабораторная работа №1
## "Основы защиты баз данных в PostgreSQL"

### Цель работы
Познакомиться с основными принципами защиты баз данных в PostgreSQL, научиться создавать пользователей с различными уровнями доступа и настраивать базовые механизмы безопасности.

### Оборудование и программное обеспечение
1. Персональный компьютер с операционной системой Windows/Linux
2. СУБД PostgreSQL (версия 11 или выше)
3. Программа pgAdmin 4 для графического управления базой данных

### Теоретические сведения

PostgreSQL предоставляет различные механизмы защиты данных:
- **Аутентификация** - процесс проверки подлинности пользователя
- **Авторизация** - определение прав доступа пользователя к объектам базы данных
- **Роли и привилегии** - система разграничения доступа к объектам БД
- **Шифрование данных** - защита конфиденциальной информации

В PostgreSQL используется ролевая модель доступа. Роль может представлять пользователя или группу пользователей. Роли могут владеть объектами базы данных и предоставлять привилегии на эти объекты другим ролям.

Основные привилегии в PostgreSQL:
- SELECT - чтение данных
- INSERT - добавление данных
- UPDATE - изменение данных
- DELETE - удаление данных
- TRUNCATE - очистка таблицы
- REFERENCES - создание внешних ключей
- TRIGGER - создание триггеров
- CREATE - создание объектов
- CONNECT - подключение к базе данных
- TEMPORARY - создание временных таблиц
- EXECUTE - выполнение функций

### Ход работы

#### Часть 1. Создание базы данных и таблиц

1. Запустите pgAdmin 4 и подключитесь к серверу PostgreSQL.

2. Создайте новую базу данных с именем `school_library`:
   - Щелкните правой кнопкой мыши на пункте "Базы данных"
   - Выберите "Создать" → "База данных..."
   - Введите имя базы данных: `school_library`
   - Нажмите "Сохранить"

3. Создайте таблицы в базе данных. Для этого откройте инструмент запросов (нажмите на значок с изображением листа бумаги и лупы) и выполните следующий SQL-скрипт:

```sql
CREATE TABLE books (
    book_id SERIAL PRIMARY KEY,
    title VARCHAR(100) NOT NULL,
    author VARCHAR(100) NOT NULL,
    publication_year INTEGER,
    available BOOLEAN DEFAULT TRUE
);

CREATE TABLE readers (
    reader_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    class VARCHAR(10) NOT NULL,
    registration_date DATE DEFAULT CURRENT_DATE
);

CREATE TABLE loans (
    loan_id SERIAL PRIMARY KEY,
    book_id INTEGER REFERENCES books(book_id),
    reader_id INTEGER REFERENCES readers(reader_id),
    loan_date DATE DEFAULT CURRENT_DATE,
    return_date DATE,
    returned BOOLEAN DEFAULT FALSE
);
```

4. Заполните таблицы данными:

```sql
INSERT INTO books (title, author, publication_year, available)
VALUES 
    ('Война и мир', 'Лев Толстой', 1869, true),
    ('Преступление и наказание', 'Федор Достоевский', 1866, true),
    ('Мастер и Маргарита', 'Михаил Булгаков', 1967, true),
    ('Евгений Онегин', 'Александр Пушкин', 1833, true),
    ('Гарри Поттер и философский камень', 'Джоан Роулинг', 1997, true);

INSERT INTO readers (first_name, last_name, class)
VALUES 
    ('Иван', 'Иванов', '10А'),
    ('Мария', 'Петрова', '10Б'),
    ('Алексей', 'Сидоров', '10А'),
    ('Екатерина', 'Смирнова', '10В');

INSERT INTO loans (book_id, reader_id, loan_date, return_date, returned)
VALUES 
    (1, 1, '2023-09-01', '2023-09-15', true),
    (2, 2, '2023-09-05', NULL, false),
    (3, 3, '2023-09-10', NULL, false);
```

#### Часть 2. Создание пользователей и настройка прав доступа

1. Создайте роли для различных типов пользователей:

```sql
-- Роль администратора библиотеки
CREATE ROLE library_admin WITH LOGIN PASSWORD 'admin123';

-- Роль библиотекаря (может просматривать и изменять данные)
CREATE ROLE librarian WITH LOGIN PASSWORD 'lib123';

-- Роль ученика (может только просматривать определенные данные)
CREATE ROLE student WITH LOGIN PASSWORD 'student123';
```

2. Предоставьте соответствующие привилегии ролям:

```sql
-- Администратор имеет полный доступ к базе данных
GRANT ALL PRIVILEGES ON DATABASE school_library TO library_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO library_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO library_admin;

-- Библиотекарь может просматривать и изменять данные во всех таблицах
GRANT SELECT, INSERT, UPDATE, DELETE ON books, readers, loans TO librarian;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO librarian;

-- Ученик может только просматривать книги и свои собственные записи о выдаче
GRANT SELECT ON books TO student;
GRANT SELECT ON readers TO student;
-- Для loans предоставим только возможность просмотра
GRANT SELECT ON loans TO student;
```

3. Создайте представление (view) для ограничения доступа учеников к данным:

```sql
-- Представление для учеников, которое показывает только доступные книги
CREATE VIEW available_books AS
SELECT book_id, title, author, publication_year
FROM books
WHERE available = true;

-- Предоставляем доступ к представлению
GRANT SELECT ON available_books TO student;
```

#### Часть 3. Настройка политики защиты на уровне строк (Row-Level Security)

1. Включите защиту на уровне строк для таблицы loans:

```sql
-- Включаем RLS для таблицы loans
ALTER TABLE loans ENABLE ROW LEVEL SECURITY;

-- Создаем политику для библиотекарей (могут видеть все записи)
CREATE POLICY librarian_all_loans ON loans
    FOR ALL
    TO librarian
    USING (true);

-- Создаем политику для учеников (могут видеть только свои записи)
CREATE POLICY student_own_loans ON loans
    FOR SELECT
    TO student
    USING (reader_id IN (SELECT reader_id FROM readers WHERE last_name = current_user));
```

#### Часть 4. Проверка настроек безопасности

1. Проверьте права доступа для роли библиотекаря:

```sql
-- Подключитесь как библиотекарь
\c school_library librarian

-- Попробуйте выполнить различные операции
SELECT * FROM books;
INSERT INTO books (title, author, publication_year, available)
VALUES ('Новая книга', 'Новый автор', 2023, true);
UPDATE books SET available = false WHERE book_id = 1;
```

2. Проверьте права доступа для роли ученика:

```sql
-- Подключитесь как ученик
\c school_library student

-- Попробуйте выполнить различные операции
SELECT * FROM available_books;
SELECT * FROM books;
SELECT * FROM loans;
-- Следующие операции должны завершиться с ошибкой
INSERT INTO books (title, author, publication_year, available)
VALUES ('Еще одна книга', 'Еще один автор', 2023, true);
UPDATE books SET available = false WHERE book_id = 2;
```

### Задание для самостоятельной работы

1. Создайте новую роль `teacher` с паролем `teacher123`.
2. Предоставьте роли `teacher` права на просмотр всех таблиц и возможность обновления статуса возврата книг в таблице `loans`.
3. Создайте представление `overdue_loans`, которое будет показывать информацию о просроченных выдачах книг (где дата возврата прошла, но книга не возвращена).
4. Предоставьте доступ к этому представлению ролям `librarian` и `teacher`.
5. Проверьте работу созданных прав доступа, подключившись под разными ролями.

### Контрольные вопросы

1. Что такое роль в PostgreSQL и чем она отличается от пользователя?
2. Какие основные привилегии можно назначать на объекты базы данных?
3. Что такое Row-Level Security и для чего она используется?
4. Какие преимущества дает использование представлений (views) с точки зрения безопасности?
5. Какие еще методы защиты данных в PostgreSQL вы знаете?

### Критерии оценки

- Оценка "5" - выполнены все задания лабораторной работы и задания для самостоятельной работы, даны правильные ответы на контрольные вопросы.
- Оценка "4" - выполнены все задания лабораторной работы, но имеются небольшие недочеты или не полностью выполнены задания для самостоятельной работы.
- Оценка "3" - выполнены только основные задания лабораторной работы.
- Оценка "2" - задания не выполнены или выполнены с грубыми ошибками.

