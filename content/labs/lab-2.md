---
title: "Лабораторна робота №2"
slug: "lab-2"
---

## Тема, Мета, Місце розташування

**Тема:** Створення бази даних у MySQL. Підключення Node.js до MySQL. Робота з ORM Sequelize.

**Мета:** Розробити та налаштувати реляційну базу даних для вебзастосунку інтернет-магазину побутової техніки. Метою є створення надійної архітектури зберігання даних, де реалізовано:
- налаштування підключення серверного середовища Node.js до СУБД MySQL;
- виконання базових CRUD-операцій за допомогою "сирих" SQL-запитів через драйвер `mysql2`;
- абстрагування роботи з даними за допомогою ORM Sequelize для спрощення коду;
- моделювання таблиць `Users` (Клієнти) та `Posts` (Відгуки клієнтів про техніку) із встановленням реляційного зв'язку One-to-Many.

**Місце розташування:**
- **GitHub (код застосунку):** [https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026](https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026)
- **Live demo (застосунок):** [https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/](https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/)

---

## Опис предметного середовища

### Актуальність теми
Будь-який сучасний інтернет-магазин потребує надійної системи зберігання інформації про користувачів та їхні відгуки. Використання реляційних баз даних (MySQL) у поєднанні з сучасними ORM-бібліотеками (Sequelize) дозволяє швидко розгортати архітектуру проєкту.

### Об'єкт та предмет роботи
- **Об'єкт:** База даних серверної частини інтернет-магазину побутової техніки.
- **Предмет:** Проєктування таблиць, налаштування зв'язків (One-to-Many) та взаємодія бекенду (Node.js) з базою даних MySQL.


---

## Хід виконання

### Крок 1: Створення бази даних у MySQL Workbench
За допомогою клієнта MySQL Workbench було розгорнуто локальний сервер. За допомогою SQL-команди `CREATE DATABASE web_backend_lab;` створено схему.

### Крок 2: Ініціалізація проєкту та встановлення залежностей
У кореневій папці бекенду було ініціалізовано Node.js проєкт (`npm init -y`) та встановлено пакети: `mysql2` та `sequelize`.

### Крок 3: Робота з "сирими" запитами (mysql2)
Створено файл `raw_queries.js`. У ньому реалізовано пряме підключення до БД та виконання чотирьох базових SQL-операцій (CRUD).

### Крок 4: Підключення ORM Sequelize та створення моделей
Створено конфігураційний файл `config/database.js` для ініціалізації Sequelize. У папці `models/` створено дві моделі: `User.js` та `Post.js`.

---

## Скріншоти

**Виконання скриптів у консолі:**
![Консоль](/assets/labs/lab-2/console-output.png)

**Структура створених таблиць у MySQL Workbench:**
![Таблиці](/assets/labs/lab-2/workbench-tables.png)

**Дані у таблиці Users:**
![Users](/assets/labs/lab-2/table-users.png)

**Дані у таблиці Posts із зовнішнім ключем UserId:**
![Posts](/assets/labs/lab-2/table-posts.png)

---

## Лістинги коду серверної частини

### Підключення до бази (config/database.js)
```javascript
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize('web_backend_lab', 'root', 'password', {   
    host: 'localhost',   
    dialect: 'mysql' 
});

module.exports = sequelize;
```

### Виконання "сирих" запитів (raw_queries.js)
```javascript
const mysql = require('mysql2/promise');

async function runRawQueries() {
    const connection = await mysql.createConnection({
        host: 'localhost',
        user: 'root',
        password: 'password', 
        database: 'web_backend_lab'
    });

    console.log("Підключено до MySQL через mysql2!");

    await connection.execute(`
        CREATE TABLE IF NOT EXISTS test_users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255),
            email VARCHAR(255)
        )
    `);

    await connection.execute("INSERT INTO test_users (name, email) VALUES ('Олексій', 'olexiy@techstore.com')");
    console.log("Дані додано (INSERT).");

    const [rows] = await connection.execute("SELECT * FROM test_users");
    console.log("Дані отримано (SELECT):", rows);

    await connection.end();
}

runRawQueries().catch(console.error);
```

### Головний файл застосунку (app.js)
```javascript
const sequelize = require('./config/database');
const User = require('./models/User');
const Post = require('./models/Post');

User.hasMany(Post); 
Post.belongsTo(User); 

async function startApp() {
    try {
        await sequelize.authenticate();
        await sequelize.sync({ force: true }); 

        const user = await User.create({
            name: 'Іван',
            email: 'ivan@gmail.com'
        });

        await Post.create({
            title: 'Відгук: Пральна машина LG',
            content: 'Дуже задоволений покупкою.',
            UserId: user.id 
        });

        const posts = await Post.findAll({ include: User });
        console.log("Дані успішно зчитані.");

    } catch (error) {
        console.error("Помилка:", error);
    }
}

startApp();
```

---

## Контрольні питання

1. **Що таке реляційна база даних?** — Це система, де дані організовані у вигляді взаємопов'язаних таблиць.
2. **Для чого використовується MySQL?** — СУБД для керування реляційними базами за допомогою SQL.
3. **Що таке ORM?** — Технологія відображення таблиць БД на об'єкти мови програмування.
4. **Для чого використовується Sequelize?** — Популярна ORM для Node.js, що спрощує роботу з MySQL.
5. **Яка різниця між hasMany та belongsTo?** — `hasMany` визначає відношення від головного запису до багатьох дочірніх, `belongsTo` — навпаки.
6. **Що таке зв’язок One-to-Many?** — Коли один рядок таблиці А пов'язаний з багатьма рядками таблиці Б.
7. **Як виконати SQL-запит з Node.js?** — Використовуючи драйвер (як-от `mysql2`) через методи `.query()` або `.execute()`.