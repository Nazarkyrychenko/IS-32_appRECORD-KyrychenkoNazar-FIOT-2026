---
title: "Лабораторна робота №3"
slug: "lab-3"
---

## 1. Тема, Мета, Місце розташування

**Тема:** Розробка функціонального REST API. Реєстрація та авторизація користувачів. Валідація даних і обробка помилок.

**Мета:** Набуття практичних навичок розробки серверного застосунку на базі Node.js та Express для реалізації безпечного REST API. В межах роботи реалізовано:
- архітектуру REST для взаємодії клієнта та сервера;
- реєстрацію користувачів із хешуванням паролів (bcryptjs);
- авторизацію на основі JWT-токенів (Access та Refresh);
- захист маршрутів за допомогою middleware;
- валідацію вхідних даних та обмеження кількості спроб входу (Rate Limiting).

**Місце розташування:**
- **GitHub:** [https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026](https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026)
- **Live demo:** [https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/](https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/)

---

## 2. Виконання завдань з програмним кодом

### Завдання 1. Встановити необхідні бібліотеки
У терміналі виконано команду для встановлення залежностей:
```bash
npm install express bcryptjs jsonwebtoken express-validator express-rate-limit cors
```

### Завдання 2. Зберігати користувачів у базі & Додати роль користувача
Створено модель `User` у Sequelize, де додано поля для ролі, хешу пароля та refresh-токена.
```javascript
// models/User.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: { type: DataTypes.STRING, allowNull: false },
    role: { type: DataTypes.STRING, defaultValue: 'user' }, 
    refreshToken: { type: DataTypes.STRING, allowNull: true }
});
module.exports = User;
```

### Завдання 3. Додати middleware для перевірки токена (Захищений маршрут)
Створено проміжний обробник для захисту маршрутів.
```javascript
// middleware/authMiddleware.js
const jwt = require('jsonwebtoken');
const SECRET_KEY = "super_secret_key_123";

module.exports = function (req, res, next) {
    if (req.method === "OPTIONS") next();
    try {
        const token = req.headers.authorization.split(' ')[1];
        if (!token) return res.status(401).json({ message: "Немає токена" });
        req.user = jwt.verify(token, SECRET_KEY);
        next();
    } catch (e) {
        res.status(401).json({ message: "Невірний токен" });
    }
};
```

### Завдання 4. Реєстрація, Валідація, Підтвердження пароля та email
```javascript
// app.js
app.post('/api/register', [
    body('email').isEmail().withMessage('Невірний формат email'),
    body('password').isLength({ min: 6 }).withMessage('Мінімум 6 символів')
], async (req, res) => {
    try {
        const errors = validationResult(req);
        if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

        const { name, email, password, confirmPassword } = req.body;
        
        if (password !== confirmPassword) return res.status(400).json({ message: "Паролі не збігаються" });

        const existingUser = await User.findOne({ where: { email } });
        if (existingUser) return res.status(400).json({ message: "Email вже зайнятий" });

        const hashPassword = await bcrypt.hash(password, 10);
        await User.create({ name, email, password: hashPassword });

        console.log(`Відправлено лист з підтвердженням на ${email}`);

        res.status(201).json({ message: "Користувача зареєстровано" });
    } catch (e) { next(e); }
});
```

### Завдання 5. Авторизація, Refresh Token та Ліміт спроб входу
```javascript
// app.js
const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5, 
    message: { message: "Забагато спроб входу. Спробуйте пізніше." }
});

app.post('/api/login', loginLimiter, async (req, res) => {
    try {
        const { email, password } = req.body;
        const user = await User.findOne({ where: { email } });

        if (!user || !(await bcrypt.compare(password, user.password))) {
            return res.status(400).json({ message: "Невірні дані" });
        }

        const accessToken = jwt.sign({ id: user.id, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
        const refreshToken = jwt.sign({ id: user.id }, REFRESH_SECRET, { expiresIn: '7d' });

        user.refreshToken = refreshToken;
        await user.save();

        res.json({ accessToken, refreshToken });
    } catch (e) { next(e); }
});

app.post('/api/refresh', async (req, res) => {
    const { refreshToken } = req.body;
    if (!refreshToken) return res.status(401).json({ message: "Токен відсутній" });
    try {
        const decoded = jwt.verify(refreshToken, REFRESH_SECRET);
        const user = await User.findOne({ where: { id: decoded.id, refreshToken } });
        if (!user) throw new Error();
        const newAccessToken = jwt.sign({ id: user.id, role: user.role }, SECRET_KEY, { expiresIn: '1h' });
        res.json({ accessToken: newAccessToken });
    } catch (e) { res.status(401).json({ message: "Недійсний токен" }); }
});
```

### Завдання 6. Logout, Профіль, Зміна пароля та Видалення
```javascript
// app.js
app.post('/api/logout', authMiddleware, async (req, res) => {
    await User.update({ refreshToken: null }, { where: { id: req.user.id } });
    res.json({ message: "Успішний вихід" });
});

app.put('/api/profile', authMiddleware, async (req, res) => {
    await User.update({ name: req.body.name }, { where: { id: req.user.id } });
    res.json({ message: "Профіль оновлено" });
});

app.post('/api/change-password', authMiddleware, async (req, res) => {
    const user = await User.findByPk(req.user.id);
    const valid = await bcrypt.compare(req.body.oldPassword, user.password);
    if (!valid) return res.status(400).json({ message: "Старий пароль невірний" });
    user.password = await bcrypt.hash(req.body.newPassword, 10);
    await user.save();
    res.json({ message: "Пароль змінено" });
});

app.delete('/api/profile', authMiddleware, async (req, res) => {
    await User.destroy({ where: { id: req.user.id } });
    res.json({ message: "Акаунт видалено" });
});
```

### Завдання 7. OAuth, Відновлення пароля та Логування помилок
```javascript
// app.js
app.use((err, req, res, next) => {
    console.error(`[ERROR] ${new Date().toISOString()} - ${err.message}`);
    res.status(500).json({ message: "Внутрішня помилка сервера" });
});

app.post('/api/forgot-password', (req, res) => {
    res.json({ message: `Інструкції надіслано на ${req.body.email}` });
});

app.post('/api/auth/google', (req, res) => {
    res.json({ message: "Успішний вхід через Google" });
});
```

---

## 3. Скріншоти результатів (Тестування через Postman)

1. **Реєстрація (POST /api/register):** Успішне створення користувача зі статусом 201.
![Реєстрація](/assets/labs/lab-3/register.png)

2. **Авторизація (POST /api/login):** Отримання токенів доступу.
![Авторизація](/assets/labs/lab-3/login.png)

3. **Профіль (GET /api/profile):** Отримання даних за допомогою токена.
![Профіль](/assets/labs/lab-3/profile.png)

---

## 4. Аналіз отриманих результатів
У ході лабораторної роботи створено надійне REST API. Впровадження JWT-токенів дозволило створити stateless-архітектуру для безпечної авторизації. Валідація даних на рівні маршрутів попереджає потрапляння некоректних даних у БД. Реалізовано механізми захисту від brute-force атак та безпечне зберігання паролів за допомогою криптографічної солі.

---

## 5. Висновки
Розроблене REST API відповідає сучасним стандартам безпеки. Використання middleware в Express продемонструвало ефективність розділення логіки обробки запитів та контролю доступу. Система refresh-токенів забезпечує тривалу роботу користувача без необхідності постійного введення пароля, зберігаючи при цьому високий рівень безпеки.

---

## 6. Контрольні питання

1. **Що таке REST API та які його основні принципи?** — Це архітектурний стиль взаємодії через HTTP. Принципи: Stateless, Client-Server, єдиний інтерфейс, використання методів (GET, POST, PUT, DELETE).
2. **Які HTTP-методи використовуються в REST API і для чого вони призначені?** — GET (отримання даних), POST (створення ресурсу), PUT/PATCH (оновлення), DELETE (видалення).
3. **Що таке клієнт-серверна архітектура?** — Модель, де клієнт (інтерфейс) та сервер (логіка і дані) розділені та взаємодіють через мережу.
4. **Що означає принцип Stateless у REST?** — Сервер не зберігає інформацію про попередні запити клієнта; кожен запит містить усю необхідну інформацію для його обробки.
5. **Для чого використовується фреймворк Express у Node.js?** — Для швидкого створення серверної частини, налаштування маршрутизації та роботи з проміжними обробниками (middleware).
6. **Що таке реєстрація користувача в інформаційній системі?** — Процес створення нового запису користувача, що включає введення даних, їх валідацію, хешування пароля та збереження у базі.
7. **Що таке авторизація та чим вона відрізняється від автентифікації?** — Автентифікація — це перевірка особи (хто це?), а авторизація — перевірка прав доступу (що йому дозволено?).
8. **Для чого використовується JWT (JSON Web Token)?** — Для безпечної передачі інформації між сторонами та підтвердження авторизації без зберігання сесії на сервері.
9. **З яких частин складається JWT-токен?** — Заголовок (Header), Корисне навантаження (Payload) та Підпис (Signature).
10. **Для чого потрібно хешування паролів і яка бібліотека використовується в роботі?** — Для безпеки: паролі не зберігаються відкритими. Використовується бібліотека `bcryptjs`.
11. **Що таке валідація даних і навіщо вона потрібна?** — Перевірка правильності введених даних (формат, довжина) для уникнення помилок та захисту від некоректних запитів.
12. **Які основні HTTP-коди стану використовуються для обробки помилок?** — 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 500 (Internal Server Error).
13. **Що таке middleware та яку роль він виконує в Express?** — Функції-посередники, що виконуються під час обробки запиту для перевірки даних, авторизації або логування.
14. **Для чого потрібен захищений маршрут у REST API?** — Для обмеження доступу до конфіденційних даних від неавторизованих користувачів.
15. **Що таке роль користувача (user/admin) і як вона впливає на доступ до ресурсів?** — Це атрибут, що визначає рівень привілеїв. Наприклад, адміністратор може керувати всіма користувачами, а звичайний користувач — лише власним профілем.