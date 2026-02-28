---
name: telethon-userbot-patterns
description: "Telethon userbot patterns for AlexSoftClub. Use when writing/modifying Telegram client code (Telethon, sessions, accounts, async tasks). Covers connection, error handling, session management, anti-flood."
risk: safe
source: project
---

# Telethon Userbot Patterns — AlexSoftClub

## Overview

Паттерны для Telethon **клиента** (userbot), НЕ Bot API. Проект использует opentele (локальная копия) для конвертации tdata → Telethon сессии.

**ВАЖНО**: Всегда сверяться с Context7 MCP (`/websites/telethon_dev_en_stable`) при работе с Telethon API. Не выдумывать методы — проверять документацию.

## Архитектура подключения

### Создание клиента (через opentele)

```python
from opentele.api import UseCurrentSession

# С прокси
client = await tdesk.ToTelethon(
    flag=UseCurrentSession,
    proxy=("SOCKS5", proxy_ip, int(port), True, login, password),
    connection_retries=60,
    auto_reconnect=True,
    timeout=30
)

# Без прокси
client = await tdesk.ToTelethon(
    flag=UseCurrentSession,
    connection_retries=60,
    auto_reconnect=True,
    timeout=30
)
```

### Подключение с таймаутом

```python
import asyncio

try:
    await asyncio.wait_for(client.connect(), timeout=10)
except asyncio.TimeoutError:
    logger.error("Тайм-аут при подключении")
except Exception as e:
    logger.error(f"Ошибка: {e}")
```

### Disconnect — ВСЕГДА async

```python
# Правильно:
await client.disconnect()

# Неправильно (устаревшее):
# client.disconnect()  # без await — ошибка "pending task destroyed"
```

## Паттерн retry с connection_reboot

Стандартный паттерн для всех операций с Telegram API:

```python
connection_reboot_nom = 0
while True:
    if connection_reboot_nom >= g.connection_reboot:
        ACCOUNT_WORKING_STATUS = "Ошибка подключения"
        ERROR = "Ошибка подключения"
        await marked_sessions(TREAD, client, ACCOUNT_WORKING_STATUS, TASK, ERROR, tread_number)
        return

    try:
        result = await client.some_api_call()
        break  # Успех — выходим из цикла

    except FloodWaitError as e:
        await flood_sessions(TREAD, client, e.seconds, TASK, tread_number)
        return

    except (ConnectionError, ProxyError, ConnectionResetError,
            ConnectionAbortedError, ConnectionRefusedError):
        ERROR = "Ошибка подключения"
        logger.info(f"Поток {tread_number}: {LOG()} {user_id}: {TASK}: {ERROR}")
        connection_reboot_nom += 1
        continue

    except Exception as e:
        logger.info(f"Поток {tread_number}: {LOG()} {user_id}: {TASK} Ошибка: {e}")
        ACCOUNT_WORKING_STATUS = "Необработанная ошибка"
        ERROR = "Необработанная ошибка"
        await marked_sessions(TREAD, client, ACCOUNT_WORKING_STATUS, TASK, ERROR, tread_number)
        return
```

## Обязательные импорты ошибок

```python
from python_socks import ProxyError
from telethon.errors import FloodWaitError
```

## Структура async-задачи (feature)

Каждая feature-функция следует шаблону:

```python
async def my_feature(TREAD, tread_number, semaphore, info_cycle_nom):
    async with semaphore:
        TASK = "Название задачи"

        # 1. Авторизация
        client, ERROR = await autorization_session(TREAD, tread_number, TASK)

        if client is not None:
            # 2. Основная логика (с retry-паттерном)
            # ...

            # 3. Маркировка результата
            ACCOUNT_WORKING_STATUS = "Выполнил"
            ERROR = None
            await marked_sessions(TREAD, client, ACCOUNT_WORKING_STATUS, TASK, ERROR, tread_number)
            return
```

## Сессии и авторизация

### Проверка авторизации

```python
if not await client.is_user_authorized():
    # Сессия неактивна — пометить и выйти
    ACCOUNT_WORKING_STATUS = "Неактивная сессия"
```

### Закрытие чужих сессий

```python
# Получить все сессии
td_result = await client.GetSessions()

# Убить все кроме текущей (hash == 0 = текущая)
for auth in td_result.authorizations:
    if auth.hash != 0:
        await client.TerminateSession(hash=auth.hash)
```

### Получение информации о пользователе

```python
from telethon.tl.functions.users import GetFullUserRequest

result = await client(GetFullUserRequest(id="me"))
phone = result.users[0].phone
username = result.users[0].username
first_name = result.users[0].first_name
premium = result.users[0].premium
bio = result.full_user.about
```

## Anti-flood правила

- `flood_sleep_threshold` — авто-слип для FloodWait < 60 сек
- При FloodWaitError > 60 сек — помечать аккаунт и возвращать управление
- Задержки между потоками настраиваются в Settings.ini
- НЕ делать retry при FloodWait — сразу `flood_sessions()` и return

## Завершение работы с клиентом

```python
from telethon.tl.functions.account import UpdateStatusRequest

# 1. Установить оффлайн статус
try:
    await client(UpdateStatusRequest(offline=True))
except Exception:
    pass

# 2. Отключиться
try:
    await client.disconnect()
except Exception:
    pass

await asyncio.sleep(1)
```

## Запрещено

- НЕ использовать `client.start()` — мы работаем с готовыми tdata сессиями
- НЕ использовать `client.run_until_disconnected()` — задачи конечные, не daemon
- НЕ использовать Bot API (`from telethon import TelegramClient` с bot_token)
- НЕ использовать `except:` (bare except) — только `except Exception:`
- НЕ вызывать `sys.exit()` при ошибках — return или raise
