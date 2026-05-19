---
title: "Лабораторна робота №5"
slug: "lab-5"
---

## 1. Тема, Мета, Місце розташування

**Тема:** Безпека та продуктивність серверних додатків. Оптимізація запитів, використання in-memory баз даних (Redis). Документування API (Swagger) та контейнеризація (Docker). Автоматизоване та навантажувальне тестування.

**Мета:**
- забезпечити комплексну безпеку Node.js-додатка (захист заголовків, обмеження запитів, валідація);
- інтегрувати зовнішнє сховище Redis для кешування даних та зменшення навантаження на БД;
- налаштувати контейнеризацію проєкту за допомогою Docker та Docker Compose;
- створити інтерактивну документацію API за стандартом OpenAPI (Swagger);
- провести інтеграційне (Jest) та навантажувальне (Artillery) тестування;
- проаналізувати продуктивність backend-застосунку.

**Місце розташування:**
- **GitHub:** [https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026](https://github.com/Nazarkyrychenko/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026)
- **Live demo:** [https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/](https://nazarkyrychenko.github.io/IS-32_appIndependent-KyrychenkoNazar-FIOT-2026/)

---

## 2. Виконання завдань з програмним кодом

### Завдання 1. Ініціалізація проєкту та встановлення залежностей
До існуючого проєкту було додано нові бібліотеки для роботи з Redis, створення документації Swagger та тестування:

```bash
npm install helmet express-rate-limit cors express-validator redis swagger-ui-express
npm install jest supertest --save-dev
```

### Завдання 2. Налаштування безпеки, кешування Redis та Swagger
У головному файлі застосунку реалізовано підключення до Redis-сервера, налаштовано middleware безпеки (Helmet, Rate Limiter) та підключено UI-документацію.

```javascript
// Фрагмент app.js
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const { body, validationResult } = require('express-validator');
const redis = require('redis');
const swaggerUi = require('swagger-ui-express');
const swaggerDocument = require('./swagger.json');

const app = express();
app.use(helmet());
app.use(express.json());

// Підключення Swagger
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));

// Підключення Redis
const redisClient = redis.createClient({ url: 'redis://redis:6379' });
redisClient.connect().then(() => console.log('Підключено до Redis успішно!'));

// Захист від DDoS (Rate Limiting)
const apiLimiter = rateLimit({
    windowMs: 1 * 60 * 1000, // 1 хвилина
    max: 20, // Максимум 20 запитів
    message: { error: "Забагато запитів! Спробуйте пізніше." }
});
app.use('/api/', apiLimiter);

// GET: Отримання товарів (з кешуванням Redis)
app.get('/api/products', async (req, res) => {
    try {
        const cachedProducts = await redisClient.get('products');
        if (cachedProducts) {
            return res.json({ source: 'redis-cache', data: JSON.parse(cachedProducts) });
        }
        // Отримання з БД (імітація)
        const productsData = [{ id: 1, name: "Laptop", price: 30000 }];
        await redisClient.setEx('products', 60, JSON.stringify(productsData));
        res.json({ source: 'database', data: productsData });
    } catch (err) {
        res.status(500).json({ message: "Помилка сервера" });
    }
});
```

### Завдання 3. Docker-контейнеризація проєкту
Для ізольованого запуску API та бази даних Redis було створено конфігурацію `docker-compose.yml`, яка об'єднує сервіси в єдину мережу.

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
    restart: always

  api:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - redis
    volumes:
      - ./logs:/app/logs
      - ./uploads:/app/uploads
```

### Завдання 4. Автоматизоване тестування API (Jest)
Написано інтеграційні тести для перевірки роботи механізмів авторизації та доступності маршрутів.

```javascript
// app.test.js
const request = require('supertest');
const app = require('./app'); // Експортований Express-додаток

describe('Інтеграційні тести API (ЛР №5)', () => {
    it('GET /api/status має бути публічним', async () => {
        const res = await request(app).get('/api/status');
        expect(res.statusCode).toBe(200);
    });

    it('GET /api/products має повертати 401 без токена (захищений роут)', async () => {
        const res = await request(app).get('/api/products');
        // Якщо маршрут налаштовано як захищений:
        expect(res.statusCode).toBe(401); 
    });
});
```

---

## 3. Скріншоти результатів (Тестування та Інфраструктура)

* **Скріншот 1:** Успішне проходження інтеграційних тестів Jest (`PASS`).
![Jest](/assets/labs/lab-5/jest-results.png)
* **Скріншот 2:** Звіт Artillery (підтвердження блокування запитів зі статусом `429 Too Many Requests`).
![Artillery](/assets/labs/lab-5/artillery-report.png)
* **Скріншот 3:** Термінал з успішним запуском контейнерів через `docker-compose up` та підключенням до БД.
![Docker](/assets/labs/lab-5/docker.png)
* **Скріншот 4:** Інтерфейс Swagger-документації (`/api-docs`) з переліком захищених та публічних маршрутів.
![Swagger](/assets/labs/lab-5/swagger.png)

---

## 4. Аналіз отриманих результатів та дослідження продуктивності

У ході дослідження продуктивності було проведено порівняння обробки GET-запитів до та після використання кешування за допомогою **Redis**. Навантажувальне тестування проводилось інструментом **Artillery** (сценарій: 50 віртуальних користувачів, по 20 запитів).

**Аналіз результатів:**
1. **Швидкодія:** Після кешування медіанний час відповіді сервера (`Median Response Time`) склав менше 1 мс, оскільки дані віддавалися з оперативної пам'яті контейнера Redis, минаючи звернення до основної бази даних.
2. **Безпека:** Artillery-тест зафіксував блокування понад 900 запитів зі статусом `429 Too Many Requests`. Це математично доводить ефективність роботи middleware `express-rate-limit`, який успішно захистив серверну інфраструктуру від перевантаження (імітації DDoS-атаки).
3. **Стабільність:** Жоден віртуальний користувач не отримав критичної помилки (`vusers.failed: 0`), що свідчить про високу стійкість архітектури.

---

## 5. Висновки

Під час виконання лабораторної роботи було створено комплексно захищений та оптимізований REST API. Впровадження `helmet` дозволило приховати вразливі HTTP-заголовки, а `express-rate-limit` створив надійний бар'єр проти brute-force та DDoS атак. Замість локального кешу було успішно інтегровано повноцінне In-memory сховище **Redis**, що значно підвищило продуктивність при високих навантаженнях. 

Увесь проєкт було успішно запаковано за допомогою **Docker**, що забезпечило кросплатформну сумісність та легкість розгортання. Для зручності клієнтської розробки впроваджено візуальну документацію **Swagger**. Тестування інструментами Jest та Artillery на практиці підтвердило заявлену надійність, стабільність та захищеність розробленої системи.

---

## 6. Контрольні запитання

1. **Що таке Helmet у Node.js?**
Це набір middleware для Express-додатків, який допомагає захистити HTTP-заголовки, приховуючи інформацію про технологічний стек (наприклад, `X-Powered-By`) та запобігаючи поширеним веб-вразливостям, таким як XSS чи clickjacking.

2. **Для чого використовується rate limiting?**
Для захисту API від надмірної кількості запитів (DDoS-атаки, brute-force перебір паролів або скрапінг). Механізм встановлює жорсткий ліміт на кількість звернень з однієї IP-адреси за визначений проміжок часу.

3. **Які переваги кешування?**
Основними перевагами є: критичне зменшення навантаження на основну базу даних (наприклад, MySQL/PostgreSQL), значне пришвидшення часу відповіді для користувача та покращення загальної пропускної здатності сервера.

4. **Чим відрізняється unit testing від integration testing?**
Unit Testing (модульне тестування) передбачає перевірку окремих функцій чи класів в ізоляції від бази даних чи мережі. Integration Testing (інтеграційне тестування, яке ми робили через Supertest) перевіряє правильність взаємодії кількох компонентів, включаючи маршрутизацію, middleware та реальну базу даних.

5. **Для чого потрібна валідація даних?**
Вона необхідна для перевірки формату та вмісту даних, які надсилає користувач, перед їх обробкою сервером. Це гарантує цілісність бази даних і захищає застосунок від ін'єкцій або непередбаченої поведінки коду.

6. **Що таке JWT?**
JWT (JSON Web Token) — це стандарт безпечної автентифікації. Токен складається з трьох частин (заголовок, корисне навантаження, підпис) і дозволяє серверу ідентифікувати користувача без необхідності зберігати його сесію в пам'яті.

7. **Які типи кешування існують?**
Існують такі види: In-memory кеш на рівні додатка (наприклад, `node-cache`), розподілене кешування (бази даних типу **Redis** або Memcached), CDN-кешування (для статичних файлів на рівні мережі доставки) та локальне кешування в браузері клієнта.

8. **Які існують способи оптимізації API?**
До основних підходів належать: кешування частих відповідей (Redis), використання пагінації для великих об'ємів даних, індексація таблиць у базі даних, стиснення відповідей (наприклад, gzip/brotli), асинхронна обробка фонових задач та горизонтальне масштабування серверів (через Docker/Kubernetes).