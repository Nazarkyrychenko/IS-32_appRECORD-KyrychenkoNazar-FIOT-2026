## Тема, Мета, Місце розташування

**Тема:** Створення бази даних у MySQL. Підключення Node.js до MySQL. Робота з ORM Sequelize.

**Мета:** Розробити та налаштувати реляційну базу даних для вебзастосунку інтернет-магазину побутової техніки. Метою є створення надійної архітектури зберігання даних, де реалізовано:
- налаштування підключення серверного середовища Node.js до СУБД MySQL;
- виконання базових CRUD-операцій (створення, читання, оновлення, видалення) за допомогою "сирих" SQL-запитів через драйвер `mysql2`;
- абстрагування роботи з даними за допомогою ORM Sequelize для спрощення коду;
- моделювання таблиць `Users` (Клієнти) та `Posts` (Відгуки клієнтів про техніку) із встановленням реляційного зв'язку One-to-Many.

**Місце розташування:**
- **GitHub (код застосунку):** [https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026]
- **Live demo (застосунок):** [https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/]

---

## Опис предметного середовища

### Актуальність теми
Будь-який сучасний інтернет-магазин потребує надійної системи зберігання інформації про користувачів та їхні відгуки. Використання реляційних баз даних (MySQL) у поєднанні з сучасними ORM-бібліотеками (Sequelize) дозволяє швидко розгортати архітектуру проєкту, забезпечувати цілісність даних та захист від базових вразливостей.

### Об'єкт та предмет роботи
- **Об'єкт:** База даних серверної частини інтернет-магазину побутової техніки.
- **Предмет:** Проєктування таблиць, налаштування зв'язків (One-to-Many) та взаємодія бекенду (Node.js) з базою даних MySQL.

### Бізнес-логіка роботи
- В інтернет-магазині один зареєстрований користувач (User) може залишити багато відгуків або запитань (Posts) до різних моделей побутової техніки.
- Кожен відгук обов'язково повинен бути прив'язаний до конкретного автора. Якщо користувач видаляється з бази, його ідентифікатор у відгуках обробляється згідно з правилами зовнішнього ключа.

### Вимоги
- **Функціональні:** створення підключення до БД, виконання сирих SQL-запитів (SELECT, INSERT, UPDATE, DELETE), програмне визначення моделей таблиць через ORM, синхронізація моделей з реальною БД.
- **Нефункціональні:** оптимізація запитів за допомогою об'єктно-реляційного відображення, автоматичне управління часовими мітками (`createdAt`, `updatedAt`).

### Use-case та ER-діаграми
![Use-case діаграма БД](/assets/labs/lab-2/use-case-db.png)

![ER-діаграма БД](/assets/labs/lab-2/er-diagram-db.png)

---

## Хід виконання

### Крок 1: Створення бази даних у MySQL Workbench
За допомогою клієнта MySQL Workbench було розгорнуто локальний сервер. За допомогою SQL-команди `CREATE DATABASE web_backend_lab;` створено порожню схему для подальшої роботи застосунку.

### Крок 2: Ініціалізація проєкту та встановлення залежностей
У кореневій папці бекенду було ініціалізовано Node.js проєкт (`npm init -y`) та встановлено необхідні пакети: драйвер для зв'язку з БД (`mysql2`) та ORM-бібліотеку (`sequelize`).

### Крок 3: Робота з "сирими" запитами (mysql2)
Створено файл `raw_queries.js`. У ньому реалізовано пряме підключення до БД та виконання чотирьох базових SQL-операцій: створення тестової таблиці та запису, оновлення цього запису, вибірка даних та їх подальше видалення.

### Крок 4: Підключення ORM Sequelize та створення моделей
Створено конфігураційний файл `config/database.js` для ініціалізації Sequelize. У папці `models/` створено дві JavaScript-моделі:
- `User.js` — містить поля `name` та `email`.
- `Post.js` — містить поля `title` та `content` (відгуки про техніку).

### Крок 5: Реалізація зв'язків та перевірка результату
У головному файлі `app.js` моделі було об'єднано зв'язком One-to-Many (`User.hasMany(Post); Post.belongsTo(User);`). За допомогою методу `.sync()` таблиці було створено у БД автоматично, після чого туди додано користувача та його відгуки. Дані успішно виведено в консоль за допомогою SQL JOIN (`include` в Sequelize) та перевірено візуально через MySQL Workbench.

---

## Скріншоти

**Виконання скриптів у консолі (raw queries та Sequelize):**
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

    await connection.execute("UPDATE test_users SET name = 'Олексій Оновлений' WHERE email = 'olexiy@techstore.com'");
    console.log("Дані оновлено (UPDATE).");

    const [rows] = await connection.execute("SELECT * FROM test_users");
    console.log("Дані отримано (SELECT):", rows);

    await connection.execute("DELETE FROM test_users WHERE email = 'olexiy@techstore.com'");
    console.log("Дані видалено (DELETE).");

    await connection.end();
}

runRawQueries().catch(console.error);
```

### Модель Користувача (models/User.js)
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false }
});

module.exports = User;
```

### Модель Поста (models/Post.js)
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Post = sequelize.define('Post', {
    title: { type: DataTypes.STRING, allowNull: false },
    content: { type: DataTypes.TEXT, allowNull: false }
});

module.exports = Post;
```

### Головний файл застосунку (app.js)
```javascript
const sequelize = require('./config/database');
const User = require('./models/User');
const Post = require('./models/Post');

// Реалізація зв'язку One-to-Many
User.hasMany(Post); 
Post.belongsTo(User); 

async function startApp() {
    try {
        await sequelize.authenticate();
        console.log("Sequelize: Підключення успішне!");

        await sequelize.sync({ force: true }); 
        console.log("Sequelize: Таблиці Users та Posts створені.");

        const user = await User.create({
            name: 'Іван',
            email: 'ivan@gmail.com'
        });

        await Post.create({
            title: 'Відгук: Пральна машина LG',
            content: 'Дуже задоволений покупкою, працює тихо.',
            UserId: user.id 
        });

        await Post.create({
            title: 'Питання: Холодильник Bosch',
            content: 'Чи є гарантія на компресор?',
            UserId: user.id
        });

        const posts = await Post.findAll({ include: User });
        
        console.log("\n--- Список постів з бази даних ---");
        posts.forEach(p => {
            console.log(`Пост: "${p.title}" | Автор: ${p.User.name} | Текст: ${p.content}`);
        });

    } catch (error) {
        console.error("Помилка:", error);
    }
}

startApp();
```

---

## Контрольні питання

**Що таке реляційна база даних?**
Це система зберігання даних, де інформація організована у вигляді пов'язаних таблиць (рядків і стовпців). Зв'язки встановлюються за допомогою первинних та зовнішніх ключів.

**Для чого використовується MySQL?**
Це система керування реляційними базами даних (RDBMS). Вона використовується для безпечного збереження, пошуку, оновлення та видалення даних за допомогою мови SQL.

**Що таке ORM?**
ORM (Object Relational Mapping) — технологія, яка відображає структуру бази даних у вигляді об’єктів програмування. Дозволяє взаємодіяти з БД через класи та методи (ООП) замість написання "сирих" SQL-запитів.

**Для чого використовується Sequelize?**
Sequelize — це популярна ORM-бібліотека для Node.js, яка спрощує роботу з реляційними базами даних (зокрема MySQL) завдяки підтримці асинхронності (Promise), автоматичній генерації запитів та зручному створенню таблиць і зв'язків.

**Яка різниця між hasMany та belongsTo?**
Метод `hasMany` застосовується до головної моделі і вказує, що вона може мати багато пов'язаних сутностей. Метод `belongsTo` застосовується до підлеглої моделі і вказує, що вона належить іншій сутності (саме в цій таблиці створюється зовнішній ключ).

**Що таке зв’язок One-to-Many?**
Це тип відношення між таблицями бази даних, коли одному запису в першій таблиці може відповідати декілька записів у другій таблиці (наприклад, один покупець може написати багато відгуків).

**Як виконати SQL-запит з Node.js?**
Для цього встановлюється драйвер (наприклад, `mysql2`). За допомогою коду створюється з'єднання (`createConnection`), після чого викликається метод `.query()` або `.execute()`, до якого передається текстовий рядок із SQL-командою.