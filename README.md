# T-Messenger

Полноценный MVP мессенджера: Flutter + NestJS + PostgreSQL + Socket.IO.

Тёмная оригинальная тема **Obsidian / Copper**. Не копия Telegram или Viber.

## Что умеет первая версия

- Регистрация и вход (email **или** телефон + пароль + username)
- JWT-авторизация, выход из аккаунта
- Профиль: id, имя, username, аватар, online/offline, last seen
- Поиск пользователей по username
- Личные чаты
- Список чатов: аватар, имя, последнее сообщение, время, непрочитанные
- Текстовые сообщения в реальном времени
- Статусы: отправлено / доставлено / прочитано
- История из PostgreSQL + пагинация
- Фото и документы
- Группы: создание, название, аватар, участники, сообщения
- Архитектура под голос/видео (кнопки звонка есть, WebRTC — следующий этап)
- Заготовки под FCM push
- Адрес сервера меняется в настройках приложения — без пересборки APK

## Структура

```
t-messenger/
  docker-compose.yml
  .env.example
  README.md
  backend/          NestJS API + WebSocket + Prisma
  mobile/           Flutter-приложение
```

## Быстрый старт backend

Нужны Docker и Docker Compose.

```bash
cd t-messenger
cp .env.example .env
docker compose up -d --build
```

API: `http://localhost:3000`
Health: `http://localhost:3000/health`
Статика загрузок: `http://localhost:3000/uploads/...`

Применить миграции (делается автоматически при старте контейнера `api`):

```bash
docker compose exec api npx prisma migrate deploy
```

Если контейнер API уже сам прогнал `prisma migrate deploy` в entrypoint — повтор не обязателен.

### Без Docker (локально)

```bash
cd backend
cp ../.env.example .env
# поправьте DATABASE_URL на localhost
npm install
npx prisma migrate deploy
npx prisma generate
npm run start:dev
```

PostgreSQL должен слушать `localhost:5432`.

## Запуск Flutter

1. Установите [Flutter](https://docs.flutter.dev/get-started/install).
2. Укажите адрес backend.

На эмуляторе Android `localhost` хоста — это `10.0.2.2`.
На реальном телефоне нужен IP компьютера в Wi‑Fi, например `192.168.0.12`, либо публичный URL.

```bash
cd mobile
flutter pub get
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:3000
```

Адрес можно сменить в приложении: **Настройки → Сервер**.

### Сборка APK

```bash
cd mobile
flutter pub get
flutter build apk --release --dart-define=API_BASE_URL=http://YOUR_PUBLIC_HOST:3000
```

Файл: `mobile/build/app/outputs/flutter-apk/app-release.apk`

Для двух телефонов через интернет backend должен быть доступен из сети.
Варианты:

- VPS + Docker Compose + открытый порт 3000 (лучше nginx + HTTPS)
- Туннель: `ngrok http 3000`, затем вставить `https://xxxx.ngrok-free.app` в Настройки → Сервер
- Домашний роутер: проброс порта на машину с Docker

## Проверка переписки на двух телефонах

1. Поднимите backend так, чтобы оба телефона видели один и тот же URL.
2. Установите APK на оба устройства.
3. На каждом телефоне откройте **Настройки → Сервер** и укажите один URL (без слэша в конце), например `http://192.168.0.12:3000` или `https://xxx.ngrok-free.app`.
4. Зарегистрируйте двух пользователей с разными username.
5. На первом телефоне: Контакты → поиск username второго → «Написать».
6. Отправьте сообщение. На втором оно должно появиться без ручного обновления.
7. Откройте чат на втором телефоне — у первого статус сменится на «прочитано».

### Тестовые аккаунты

Можно создать через UI или скрипт:

```bash
cd backend
node scripts/seed-users.js
```

По умолчанию:

| Username | Пароль    | Email             |
|----------|-----------|-------------------|
| timur    | Test1234! | timur@tmsg.local  |
| eva      | Test1234! | eva@tmsg.local    |

Скрипт идемпотентный: повторный запуск не ломает существующих пользователей.

## Переменные окружения

См. `.env.example`.

Секреты не хранятся в коде. Для продакшена обязательно смените `JWT_SECRET`.

## Firebase Cloud Messaging

Полноценная отправка push зависит от вашего Firebase-проекта.

1. Создайте проект в Firebase Console.
2. Добавьте Android-приложение с package `com.tmessenger.app`.
3. Скачайте `google-services.json` в `mobile/android/app/`.
4. Сервис-аккаунт JSON положите на сервер, путь укажите в `FIREBASE_SERVICE_ACCOUNT`.
5. Клиент регистрирует FCM-токен через `POST /users/me/push-token`.
6. Сервер шлёт notification при `new_message`, если получатель offline.

Пока файл сервис-аккаунта не задан, backend логирует push как skipped — чаты при этом работают.

## WebRTC-звонки

В UI есть кнопки аудио/видео. Нажатие показывает, что слой звонков готовится.
На backend зарезервированы события сокета:

- `call_offer` / `call_answer` / `call_ice` / `call_hangup`

Их можно подключить к WebRTC без ломки текущей модели чатов.

## API (кратко)

- `POST /auth/register`
- `POST /auth/login`
- `GET /users/me`
- `PATCH /users/me`
- `GET /users/search?q=`
- `GET /users/:id`
- `GET /chats`
- `POST /chats/private`
- `POST /chats/group`
- `GET /chats/:id`
- `GET /chats/:id/messages?cursor=&limit=`
- `POST /chats/:id/messages`
- `POST /messages/:id/read`
- `POST /uploads`
- `GET /health`

Авторизация: заголовок `Authorization: Bearer <accessToken>`.

## WebSocket

URL: тот же origin, path `/socket.io`.

Handshake auth: `auth: { token }` или `Authorization: Bearer`.

События:

- `join_chat` `{ chatId }`
- `leave_chat` `{ chatId }`
- `send_message` `{ chatId, type, text, attachmentId? }`
- `new_message`
- `message_delivered`
- `message_read`
- `typing_start` / `typing_stop`
- `user_online` / `user_offline`

## Безопасность

- Пароли только bcrypt
- JWT на REST и на сокете
- Доступ только к своим чатам
- class-validator на DTO
- Лимит файла 10 МБ, белый список MIME
- Секреты в `.env`

## Лицензия

Учебный MVP. Используйте на свой риск.
