vk1.a.O9BEe52I1uH-jGc8E_XFoAgGfsqU5O478rH7tJ-dHA3lNGmSfANibTsvgeV6cPJHZWipYwFYCRuTCVY7y8yBUfwgJUB62WibeXB0-mr86ZZqW8e96bC7CZ3zFUq-ArvyzvEvKvlnXQpU7itizdnn0bCovWlQ07Z7xfGVfc2wbL2lvgQRnDOr2QfGx5le4JhUBFIpW0RYXn-gRlCAXP2faQ

# Полное руководство: создание бота ВКонтакте для сообщества с нуля

В этом руководстве мы пройдём путь от регистрации сообщества до работающего бота с полноценным кодом. Мы будем использовать Python и библиотеку `vk_api`, которая является стандартом де-факто для работы с VK API на Python. В конце вы получите готовый скрипт с обработкой команд, клавиатурой и базовой логикой.


## Шаг 1. Подготовка сообщества и получение токена

Прежде чем писать код, нужно настроить сообщество ВКонтакте.

1. **Создайте сообщество** (публичную страницу или группу) от имени, под которым будет работать бот.
2. Перейдите в **Управление** → **Работа с API**.
3. Нажмите **Создать ключ доступа**. Выберите права: как минимум `Управление сообществом` и `Сообщения сообщества`. Сохраните полученный **токен** — он понадобится в коде.
4. Перейдите в **Управление** → **Сообщения** и включите **Сообщения сообщества**. Это обязательно, иначе бот не сможет получать и отправлять сообщения.
5. Там же, в разделе **Настройки для бота**, включите **Возможности ботов** и **Разрешить добавлять сообщество в беседы** (если нужно).

Теперь у вас есть токен и ID сообщества (его видно в адресе страницы или в настройках).


## Шаг 2. Установка библиотек и структура проекта

Для работы нам понадобятся две библиотеки:

```bash
pip install vk_api python-dotenv
```

`vk_api` — официальная Python-обёртка для VK API . `python-dotenv` позволит хранить токен в файле `.env` и не светить его в коде.

Создайте папку проекта со следующей структурой:

```
vk_bot/
├── .env
├── bot.py
└── requirements.txt
```

Файл `.env`:

```env
VK_TOKEN=ваш_токен_из_настроек_сообщества
GROUP_ID=123456789
```

Файл `requirements.txt`:

```
vk_api
python-dotenv
```


## Шаг 3. Минимальный бот: приём и отправка сообщений

Начнём с самого простого: бот подключается к Long Poll и отвечает на любое сообщение.

Long Poll — это механизм, при котором бот сам опрашивает сервер ВК на предмет новых событий. Для сообществ используется `VkBotLongPoll` .

```python
# bot.py — минимальная версия
import os
import vk_api
from vk_api.bot_longpoll import VkBotLongPoll, VkBotEventType
from dotenv import load_dotenv

load_dotenv()

TOKEN = os.getenv('VK_TOKEN')
GROUP_ID = int(os.getenv('GROUP_ID'))

# Авторизация
vk_session = vk_api.VkApi(token=TOKEN)
vk = vk_session.get_api()

# Long Poll для сообщества
longpoll = VkBotLongPoll(vk_session, GROUP_ID)

print('Бот запущен. Ожидание сообщений...')

for event in longpoll.listen():
    if event.type == VkBotEventType.MESSAGE_NEW:
        # event.obj.message — объект сообщения
        msg = event.obj.message
        user_id = msg['from_id']
        text = msg['text']

        print(f'Получено от {user_id}: {text}')

        # Отправляем ответ
        vk.messages.send(
            peer_id=user_id,
            message=f'Вы написали: {text}',
            random_id=0  # обязательный параметр, можно 0
        )
```

**Что здесь происходит:**
- `VkApi(token=TOKEN)` — создаёт сессию с правами бота сообщества.
- `VkBotLongPoll` — слушает события именно для сообщества (в отличие от `VkLongPoll`, который работает от имени пользователя).
- `event.obj.message` — словарь с полями `from_id`, `text`, `peer_id` и другими .
- `random_id=0` — обязательный параметр для `messages.send`, чтобы ВК не отправил сообщение дважды (можно использовать `random.randint(1, 2**31)`).


## Шаг 4. Обработка команд и клавиатура

Теперь научим бота понимать команды и показывать клавиатуру.

```python
# bot.py — команды и клавиатура
import os
import random
import vk_api
from vk_api.bot_longpoll import VkBotLongPoll, VkBotEventType
from vk_api.keyboard import VkKeyboard, VkKeyboardColor
from dotenv import load_dotenv

load_dotenv()

TOKEN = os.getenv('VK_TOKEN')
GROUP_ID = int(os.getenv('GROUP_ID'))

vk_session = vk_api.VkApi(token=TOKEN)
vk = vk_session.get_api()
longpoll = VkBotLongPoll(vk_session, GROUP_ID)


def get_main_keyboard():
    """Создаёт клавиатуру с кнопками."""
    keyboard = VkKeyboard(one_time=False)  # False — клавиатура не исчезает после нажатия
    keyboard.add_button('Привет', color=VkKeyboardColor.PRIMARY)
    keyboard.add_button('Помощь', color=VkKeyboardColor.SECONDARY)
    keyboard.add_line()  # перенос на новую строку
    keyboard.add_button('Случайное число', color=VkKeyboardColor.POSITIVE)
    return keyboard.get_keyboard()


def send_message(peer_id, text, keyboard=None):
    """Универсальная функция отправки."""
    params = {
        'peer_id': peer_id,
        'message': text,
        'random_id': random.randint(1, 2**31)
    }
    if keyboard:
        params['keyboard'] = keyboard
    vk.messages.send(**params)


print('Бот запущен.')

for event in longpoll.listen():
    if event.type != VkBotEventType.MESSAGE_NEW:
        continue

    msg = event.obj.message
    user_id = msg['from_id']
    text = msg['text'].strip().lower()

    if text == 'привет' or text == 'начать':
        send_message(
            user_id,
            'Привет! Я бот этого сообщества. Выбери действие на клавиатуре.',
            get_main_keyboard()
        )

    elif text == 'помощь':
        send_message(
            user_id,
            'Доступные команды:\n'
            '• Привет — показать клавиатуру\n'
            '• Случайное число — сгенерировать число от 1 до 100\n'
            '• Помощь — это сообщение'
        )

    elif text == 'случайное число':
        number = random.randint(1, 100)
        send_message(user_id, f'Твоё случайное число: {number}')

    else:
        send_message(
            user_id,
            'Я не понимаю эту команду. Напиши "Помощь" или нажми кнопку.',
            get_main_keyboard()
        )
```

**Ключевые моменты:**
- `VkKeyboard(one_time=False)` — клавиатура остаётся видимой после нажатия. Если `True` — исчезает.
- `VkKeyboardColor.PRIMARY` (синяя), `SECONDARY` (белая), `POSITIVE` (зелёная), `NEGATIVE` (красная).
- Клавиатура передаётся в `messages.send` как строка JSON, которую возвращает `keyboard.get_keyboard()`.


## Шаг 5. Полноценный бот с функциями

Теперь соберём всё вместе и добавим больше возможностей: обработку вложений, работу с беседами, логирование и структурированный код.

```python
# bot.py — полноценная версия
import os
import random
import logging
import vk_api
from vk_api.bot_longpoll import VkBotLongPoll, VkBotEventType
from vk_api.keyboard import VkKeyboard, VkKeyboardColor
from dotenv import load_dotenv

# ---------- Настройка логирования ----------
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# ---------- Загрузка конфигурации ----------
load_dotenv()
TOKEN = os.getenv('VK_TOKEN')
GROUP_ID = int(os.getenv('GROUP_ID'))

if not TOKEN or not GROUP_ID:
    raise ValueError('Проверьте .env: VK_TOKEN и GROUP_ID должны быть заданы')

# ---------- Инициализация ----------
vk_session = vk_api.VkApi(token=TOKEN)
vk = vk_session.get_api()
longpoll = VkBotLongPoll(vk_session, GROUP_ID)


# ---------- Клавиатуры ----------
def main_keyboard():
    kb = VkKeyboard(one_time=False)
    kb.add_button('👋 Привет', color=VkKeyboardColor.PRIMARY)
    kb.add_button('ℹ️ Помощь', color=VkKeyboardColor.SECONDARY)
    kb.add_line()
    kb.add_button('🎲 Случайное число', color=VkKeyboardColor.POSITIVE)
    kb.add_button('📊 Статистика', color=VkKeyboardColor.PRIMARY)
    return kb.get_keyboard()


def back_keyboard():
    kb = VkKeyboard(one_time=False)
    kb.add_button('🔙 Назад', color=VkKeyboardColor.NEGATIVE)
    return kb.get_keyboard()


# ---------- Утилиты ----------
def send_message(peer_id, text, keyboard=None, attachment=None):
    """Отправляет сообщение с возможностью прикрепить вложение."""
    params = {
        'peer_id': peer_id,
        'message': text,
        'random_id': random.randint(1, 2**31)
    }
    if keyboard:
        params['keyboard'] = keyboard
    if attachment:
        params['attachment'] = attachment
    try:
        vk.messages.send(**params)
        logger.info(f'Отправлено {peer_id}: {text[:50]}...')
    except Exception as e:
        logger.error(f'Ошибка отправки {peer_id}: {e}')


def get_user_name(user_id):
    """Получает имя пользователя по ID."""
    try:
        users = vk.users.get(user_ids=user_id)
        if users:
            return f"{users[0]['first_name']} {users[0]['last_name']}"
    except Exception:
        pass
    return f'Пользователь {user_id}'


# ---------- Обработчики ----------
def handle_start(user_id, peer_id):
    name = get_user_name(user_id)
    send_message(
        peer_id,
        f'Привет, {name}! 👋\n\n'
        'Я бот сообщества. Умею показывать клавиатуру, '
        'генерировать числа и отвечать на команды.\n\n'
        'Выбери действие:',
        main_keyboard()
    )


def handle_help(peer_id):
    send_message(
        peer_id,
        '📖 **Доступные команды**\n\n'
        '• 👋 Привет — приветствие и клавиатура\n'
        '• ℹ️ Помощь — это сообщение\n'
        '• 🎲 Случайное число — число от 1 до 100\n'
        '• 📊 Статистика — информация о сообществе\n'
        '• 🔙 Назад — вернуться в главное меню\n\n'
        'Также я понимаю текстовые команды: "привет", "помощь", '
        '"случайное число", "статистика".',
        back_keyboard()
    )


def handle_random(peer_id):
    number = random.randint(1, 100)
    send_message(
        peer_id,
        f'🎲 Твоё случайное число: **{number}**\n\n'
        'Нажми кнопку ещё раз, чтобы сгенерировать новое.',
        main_keyboard()
    )


def handle_stats(peer_id):
    try:
        # Получаем информацию о сообществе
        group_info = vk.groups.getById(group_id=GROUP_ID, fields='members_count')[0]
        members = group_info.get('members_count', 'неизвестно')
        send_message(
            peer_id,
            f'📊 **Статистика сообщества**\n\n'
            f'👥 Участников: {members}\n'
            f'🤖 Бот работает стабильно\n'
            f'🕐 Время сервера: {__import__("datetime").datetime.now().strftime("%H:%M:%S")}',
            main_keyboard()
        )
    except Exception as e:
        logger.error(f'Ошибка статистики: {e}')
        send_message(peer_id, 'Не удалось получить статистику 😔', main_keyboard())


def handle_unknown(peer_id, text):
    send_message(
        peer_id,
        f'🤔 Я не понимаю команду «{text}».\n\n'
        'Напиши "Помощь" или выбери действие на клавиатуре.',
        main_keyboard()
    )


# ---------- Главный цикл ----------
def main():
    logger.info('Бот запущен. Ожидание событий...')

    for event in longpoll.listen():
        try:
            # Обрабатываем только новые сообщения
            if event.type != VkBotEventType.MESSAGE_NEW:
                continue

            msg = event.obj.message
            user_id = msg['from_id']
            peer_id = msg['peer_id']  # peer_id может отличаться от from_id в беседах
            text = msg.get('text', '').strip().lower()

            # Если сообщение из беседы — peer_id будет ID беседы (2000000000+)
            # Отвечаем туда же, куда пришло сообщение
            logger.info(f'Сообщение от {user_id} (peer={peer_id}): {text}')

            # Маршрутизация команд
            if text in ('начать', 'start', 'привет', '👋 привет'):
                handle_start(user_id, peer_id)
            elif text in ('помощь', 'help', 'ℹ️ помощь'):
                handle_help(peer_id)
            elif text in ('случайное число', '🎲 случайное число', 'рандом'):
                handle_random(peer_id)
            elif text in ('статистика', '📊 статистика'):
                handle_stats(peer_id)
            elif text in ('назад', '🔙 назад'):
                handle_start(user_id, peer_id)
            else:
                handle_unknown(peer_id, text)

        except Exception as e:
            logger.exception(f'Критическая ошибка в обработчике: {e}')
            # Пытаемся уведомить пользователя, если возможно
            try:
                send_message(
                    event.obj.message['peer_id'],
                    '⚠️ Произошла ошибка. Попробуйте позже.'
                )
            except Exception:
                pass


if __name__ == '__main__':
    main()
```

**Разбор ключевых улучшений:**

| Функция | Назначение |
|---|---|
| `logging` | Все действия и ошибки записываются в консоль с временем — удобно для отладки |
| `peer_id` вместо `user_id` | В беседах `peer_id` = `2000000000 + chat_id`. Отправка на `peer_id` гарантирует, что ответ придёт в тот же чат |
| `get_user_name` | Персонализация приветствия |
| `random_id` с `random.randint` | Более надёжно, чем `0`, хотя ВК принимает и `0` |
| `try/except` вокруг всего | Бот не упадёт от одной ошибки |
| `groups.getById` | Пример запроса к API за данными сообщества |


## Шаг 6. Запуск и отладка

1. Убедитесь, что в настройках сообщества (**Управление → Сообщения**) включены сообщения и бот не заблокирован.
2. Запустите скрипт:
   ```bash
   python bot.py
   ```
3. Напишите сообществу от своего аккаунта. Если бот не отвечает:
   - Проверьте, что токен действителен и имеет права `messages`.
   - Проверьте, что `GROUP_ID` — число, а не строка с URL.
   - Посмотрите логи в консоли.

**Важно:** Long Poll требует, чтобы у сообщества был включён Bots Long Poll. Если при запуске возникает ошибка `Access denied`, проверьте, что в настройках сообщества (**Работа с API → Long Poll API**) включён **Long Poll API** и выбрана версия **5.131** или выше.

- **Состояния (FSM)** — для многошаговых диалогов (например, оформление заказа).

Готовый код из Шага 5 можно сразу использовать как основу для реального бота. Он обрабатывает команды, работает в беседах, логирует ошибки и не падает при сбоях.
