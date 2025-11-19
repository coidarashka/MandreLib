# MandreLib - Полная документация для разработчиков

*Документация обновлена для версии 1.7*

---

## Что нового в версии 1.7

### 🚀 Основные изменения

- **Новый класс `MandreInstall`** — единый и удобный интерфейс для установки Python пакетов
- **MandreSettings DSL** — HTML-подобный язык разметки для создания настроек плагинов
- **MandreSuggestions** — кастомные меню автодополнения (как @ или / в Telegram)
- **MandreMessages** — мгновенный доступ к сообщениям из локальной SQLite базы
- **MandreSend** — прямая отправка изображений в текущий чат
- **Превью чата в настройках** — визуализация сообщений прямо в UI
- **TARGETlang** — нативная поддержка многоязычности плагинов
- **BottomSheet UI** — современные нижние панели с DSL
- **Улучшенная локализация** — автоматический перевод через Pollinations.AI

---

## 🎯 Быстрый старт

```python
from mandre_lib import Mandre, MandreData, MandreUI, MandreTTS, MandreAuth, MandreShare, MandreDevice, MandreNotification, MandrePip, MandreWeb, MandreSend, MandreInstall, MandreSettings, MandreSuggestions, MandreMessages
from base_plugin import BasePlugin

class MyPlugin(BasePlugin):
    TARGETlang = "ru"  # Укажите язык плагина
    
    def on_plugin_load(self):
        # Активируем персистентное хранилище
        Mandre.use_persistent_storage(self)
        
        # Убеждаемся, что PIP готов
        MandreInstall("check_status")  # или MandreInstall(["requests", "pillow"])
        
        # Регистрируем команды
        Mandre.register_command(self, "test", self.cmd_test)
        
        # Настраиваем автодополнение
        Mandre.Suggestions.register(self, "@mybot", self.create_suggestions_menu())
        
        self.log("Плагин загружен, MandreLib 1.7 активирована")
```

---

## 0. Установка Python пакетов (MandreInstall) ⭐ НОВОЕ

### Быстрая установка

```python
# Установить один пакет
MandreInstall("requests")

# Установить несколько пакетов
MandreInstall(["pyrogram", "aiohttp"])

# Установить из URL
MandreInstall("https://files.pythonhosted.org/.../package.whl")

# Проверить статус PIP
if MandreInstall("check_status"):
    print("PIP готов к работе")
```

### Установка из ZIP архива

```python
# Скачивает и автоматически устанавливает все .whl файлы из архива
MandreInstall("https://example.com/libs.zip")
```

### Низкоуровневые операции (MandrePip)

```python
# Выполнить произвольную pip команду
code, out, err = Mandre.Pip.pip(["list", "--outdated"])

# Получить путь к site-packages
site_dir = Mandre.Pip.site_dir()

# Установить с опциями
Mandre.Pip.install("colorama==0.4.6")

# Импортировать модуль после установки
try:
    import requests
except ImportError:
    requests = Mandre.Pip.import_module("requests")
```

### Кастомные настройки PIP

Добавьте в `create_settings()`:

```python
Input(
    key="pip_index_url", 
    text="Кастомный PyPI индекс",
    subtext="Например: https://pypi.org/simple/",
    default=""
),
Input(
    key="pip_proxy", 
    text="HTTP прокси",
    subtext="Формат: http://user:pass@host:port",
    default=""
),
Switch(
    key="pip_prefer_binary",
    text="Предпочитать бинарные пакеты",
    default=True
)
```

---

## 1. Персистентное хранилище данных (MandreData)

### Автоматическое сохранение настроек

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Все set_setting() теперь автоматически сохраняются
    count = self.get_setting("counter", 0)
    self.set_setting("counter", count + 1)
    self.log(f"Плагин загружался {count + 1} раз")
```

### Работа с JSON файлами

```python
# Записать данные
my_data = {"users": [{"id": 1, "name": "Alice"}], "settings": {"theme": "dark"}}
MandreData.write_persistent_json(self.id, "database.json", my_data)

# Прочитать данные
loaded_data = MandreData.read_persistent_json(self.id, "database.json", default={})

# Список файлов плагина
files = MandreData.list_files_for_plugin(self.id)

# Удалить все данные плагина
MandreData.delete_persistent_plugin_data(self.id)
```

### Импорт/Экспорт данных

```python
# Экспорт в ZIP
def export_plugin_data(self):
    downloads_dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    zip_filename = f"mandrelib_data_{self.id}_{timestamp}.zip"
    zip_path = os.path.join(downloads_dir.getAbsolutePath(), zip_filename)

    with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
        for filename in MandreData.list_files_for_plugin(self.id):
            file_path = MandreData.get_persistent_path(self.id, filename)
            zipf.write(file_path, arcname=filename)
    
    BulletinsHelper.show_success(f"Данные экспортированы в Downloads/{zip_filename}")

# Импорт из ZIP
def import_plugin_data(self, zip_path):
    MandreData.delete_persistent_plugin_data(self.id)
    target_dir = File(MandreData._get_base_data_dir(), self.id)
    if not target_dir.exists(): target_dir.mkdirs()
    
    with zipfile.ZipFile(zip_path, 'r') as zipf:
        zipf.extractall(target_dir.getAbsolutePath())
    
    BulletinsHelper.show_success("Данные импортированы!")
```

---

## 2. UI-компоненты (MandreUI)

### Простой диалог с выбором

```python
MandreUI.show(
    title="Выберите действие",
    items=["Первое", "Второе", "Третье"],
    on_select=lambda index, text: self.log(f"Выбран пункт {index}: {text}"),
    message="Какой вариант тебе нравится?",
    cancel_text="Отмена"
)
```

### Ripple эффект

```python
# Показать волну в центре экрана
MandreUI.ripple(intensity=2.0, vibrate=True)
```

### Селектор чатов

```python
def select_chat_handler(chat_info):
    # chat_info содержит: 'title', 'id', 'obj'
    self.log(f"Выбран чат: {chat_info['title']} (ID: {chat_info['id']})")

MandreUI.select_chat(
    title="Выберите чат",
    on_select=select_chat_handler,
    search_hint="Поиск чата...",
    cancel_text="Отмена"
)
```

### BottomSheet с DSL ⭐ НОВОЕ

```python
ui_dsl = """
<sheet title="Мой BottomSheet" title_size="24" close_text="Закрыть" close_size="16">
    <tag text="Важно" color="#E6E6FA" size="10" />
    <subtext>Это пример современного UI</subtext>
    
    <content size="18">
        Основной контент здесь.
        Можно использовать **жирный**, *курсив*.
    </content>
    
    <actions>
        <button id="save" text="Сохранить" icon="msg_check" />
        <button id="share" text="Поделиться" icon="msg_share" />
        
        <menu icon="msg_more">
            <item id="delete" text="Удалить" icon="msg_delete" />
            <item id="info" text="Информация" icon="msg_info" />
        </menu>
    </actions>
</sheet>
"""

callbacks = {
    "save": lambda: self.save_data(),
    "share": lambda: Mandre.Share.share_text("Поделился!"),
    "delete": lambda: self.delete_data(),
    "info": lambda: BulletinHelper.show_info("Информация")
}

MandreUI.show_bottom_sheet(self, ui_dsl, callbacks)
```

### Навигационная панель в настройках (BottomBar)

```python
from android.graphics import Color

def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    items = [
        {
            "text": "Основные",
            "icon": "msg_settings",
            "on_click": lambda: self._handle_tab(0)
        },
        {
            "text": "Данные",
            "icon": "msg_storage",
            "on_click": lambda: self._handle_tab(1)
        },
    ]
    
    MandreUI.setup_settings_bottom_bar(
        plugin_instance=self,
        items=items,
        active_index_key="current_tab",
        bg_color=Color.argb(210, 50, 50, 55),
        active_color=Color.WHITE,
        inactive_color=Color.rgb(140, 140, 140),
    )

def _handle_tab(self, index: int):
    self.set_setting("current_tab", index)
    Mandre.apply_and_refresh_settings(self)

def create_settings(self):
    tab = self.get_setting("current_tab", 0)
    if tab == 0:
        return [Header(text="Основные параметры")]
    else:
        return [Header(text="Управление данными")]
```

---

## 3. Рендеринг HTML в изображение (MandreWeb)

```python
html = """
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial; padding: 20px; background: #1a1a1a; color: white; }
        .card { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); padding: 30px; border-radius: 15px; }
        h1 { text-align: center; }
    </style>
</head>
<body>
    <div class="card">
        <h1>📊 Отчёт</h1>
        <p>Время: <script>document.write(new Date().toLocaleString('ru-RU'))</script></p>
    </div>
</body>
</html>
"""

def on_result(success: bool, result_path: str):
    if success:
        MandreSend.png(result_path, "📊 Сгенерированный отчёт")
    else:
        BulletinHelper.show_error(f"Ошибка: {result_path}")

MandreWeb.render_html_to_png(
    html=html,
    on_result=on_result,
    width=1024,
    height=768,
    bg_color=(26, 30, 36),
    file_prefix="report_"
)
```

---

## 4. Отправка файлов и текста (MandreShare & MandreSend)

### Отправка текста через меню "Поделиться"

```python
Mandre.Share.share_text(
    text="Привет! Это сообщение от моего плагина!",
    title="Поделиться приветствием"
)
```

### Отправка файлов

```python
# Отправить документ
Mandre.Share.share_file(
    file_path="/path/to/document.pdf",
    title="Поделиться документом"
)

# Создать и отправить временный файл
import tempfile
with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
    f.write("Содержимое файла\n")
    temp_path = f.name

Mandre.Share.share_file(temp_path, "Отправить текстовый файл")
# Файл удалится автоматически через 5 минут
```

### Прямая отправка в чат (MandreSend) ⭐ НОВОЕ

```python
# Отправить PNG изображение напрямую
MandreSend.png("/path/to/image.png", "📸 Изображение из плагина")

# Отправить график из HTML
def on_ready(success, path):
    if success:
        MandreSend.png(path, "📈 График продаж")

MandreWeb.render_html_to_png(html_chart, on_ready)
```

---

## 5. Информация об устройстве (MandreDevice)

```python
# Полная информация
device_info = Mandre.Device.get_device_info()
self.log(f"Производитель: {device_info['manufacturer']}")
self.log(f"Root: {'✅' if device_info['is_rooted'] else '❌'}")
self.log(f"Эмулятор: {'⚠️' if device_info['is_emulator'] else '✅'}")

# Краткая строка
simple = Mandre.Device.get_simple_info()
# "Samsung Galaxy S21 (Android 13, API 33)"

# Создать отчёт
def create_device_report(self):
    info = MandreDevice.get_device_info()
    report = f"""
📱 ПОДРОБНАЯ ИНФОРМАЦИЯ ОБ УСТРОЙСТВЕ

🏭 Производитель: {info.get('manufacturer', 'Unknown')}
📱 Модель: {info.get('model', 'Unknown')}
🤖 Android: {info.get('android_version', 'Unknown')} (API {info.get('api_level', 'Unknown')})
💾 Память: {info.get('total_memory_mb', 'Unknown')} МБ
🔋 Заряд: {self._get_battery_level() or 'Unknown'}%
🔒 Root: {'✅ Есть' if info.get('is_rooted') else '❌ Нет'}
⚠️ Эмулятор: {'✅ Да' if info.get('is_emulator') else '❌ Нет'}
"""
    Mandre.Share.share_text(report, "📱 Отчёт об устройстве")
```

---

## 6. Системные уведомления (MandreNotification)

### Простое уведомление

```python
Mandre.Notification.show_simple(
    title="MandreLib Demo",
    text="Это простое уведомление! 🚀",
    channel_id="my_plugin_notifications"
)
```

### Уведомление в стиле диалога

```python
Mandre.Notification.show_dialog(
    sender_name="Мой Плагин",
    message="Привет! Это уведомление в стиле диалога!",
    avatar_url="https://example.com/avatar.png",
    channel_id="my_plugin_dialogs"
)
```

### Система напоминаний

```python
def schedule_reminder(self, text: str, delay_seconds: int):
    def send_reminder():
        Mandre.Notification.show_dialog(
            sender_name="Напоминание",
            message=f"⏰ {text}",
            avatar_url="https://i.postimg.cc/436ngppG/image.png",
            channel_id="plugin_reminders"
        )
        MandreTTS.speak(f"Напоминание: {text}")

    task_name = f"reminder_{int(time.time())}"
    Mandre.schedule_task(self, task_name, delay_seconds, send_reminder)
    BulletinHelper.show_success(f"Напоминание установлено на {delay_seconds} сек")
```

---

## 7. Текст-в-речь (MandreTTS)

```python
def speak_hello(self):
    MandreTTS.speak("Привет, мир!")

def on_send_message_hook(self, params):
    message = params.get("message", "")
    if message.startswith(".say "):
        text = message[5:]
        MandreTTS.speak(text)
        return HookResult(strategy=HookStrategy.CANCEL)
    return HookResult()
```

**Важно**: Первый вызов может быть медленным (инициализация движка).

```python
def safe_speak(self, text: str):
    try:
        if not text.strip():
            return
        
        if not hasattr(_TTS_STATE, 'init_ok') or not _TTS_STATE.init_ok:
            BulletinHelper.show_info("Подготавливаю синтезатор речи...")
        
        MandreTTS.speak(text)
    except Exception as e:
        self.log(f"Ошибка TTS: {e}")
        BulletinHelper.show_error("Не удалось озвучить текст")
```

---

## 8. Аутентификация (MandreAuth)

```python
def request_auth(self):
    def on_auth_success():
        self.log("Пользователь подтвердил личность!")
        BulletinHelper.show_success("Доступ разрешён")
        self._perform_secure_action()
    
    def on_auth_failure():
        self.log("Аутентификация не удалась")
        BulletinHelper.show_error("Доступ запрещён")
    
    MandreAuth.request(
        on_success=on_auth_success,
        on_failure=on_auth_failure,
        title="Требуется подтверждение",
        description="Подтвердите свою личность для доступа к настройкам безопасности"
    )

def delete_all_data(self):
    # Защита критической операции
    MandreAuth.request(
        on_success=lambda: (
            MandreData.delete_persistent_plugin_data(self.id),
            BulletinHelper.show_success("Все данные удалены!")
        ),
        on_failure=lambda: BulletinHelper.show_info("Удаление отменено"),
        title="Подтверждение удаления",
        description="Введите пароль/код разблокировки"
    )
```

---

## 9. Планировщик задач (Scheduler)

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Запустить задачу каждые 30 секунд
    Mandre.schedule_task(
        self, 
        task_name="auto_check",
        interval_seconds=30,
        callback=self.check_something
    )

def check_something(self):
    self.log("Проверка выполнена")

def on_plugin_unload(self):
    # Отменить задачу при выгрузке
    Mandre.cancel_task(self, "auto_check")
```

### Продвинутое использование

```python
def setup_monitoring(self):
    # Мониторинг системы
    Mandre.schedule_task(self, "system_monitor", 60, self.monitor_system)
    
    # Очистка временных файлов
    Mandre.schedule_task(self, "cleanup_temp", 300, self.cleanup_temp_files)

def monitor_system(self):
    device_info = MandreDevice.get_device_info()
    if device_info.get("available_memory_mb", 0) < 100:
        Mandre.Notification.show_simple(
            "Предупреждение",
            "Мало свободной памяти. Рекомендуется очистить кэш.",
            channel_id="memory_warnings"
        )

def cleanup_temp_files(self):
    temp_dir = "/data/data/com.exteragram.messenger/cache"
    # Очистка файлов старше часа...
```

---

## 10. Команды (Commands)

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    self.add_on_send_message_hook()
    
    # Регистрируем команды
    Mandre.register_command(self, "hello", self.cmd_hello)
    Mandre.register_command(self, "echo", self.cmd_echo)
    Mandre.register_command(self, "device", self.cmd_device)
    Mandre.register_command(self, "notify", self.cmd_notify)

def on_send_message_hook(self, params):
    # MandreLib автоматически обрабатывает команды
    return Mandre.handle_outgoing_command(params) or HookResult()

def cmd_hello(self, plugin, args, params):
    """Использование: .hello"""
    BulletinHelper.show_info("Привет!")
    return None  # Сообщение не изменится, но будет отменено

def cmd_echo(self, plugin, args, params):
    """Использование: .echo текст"""
    if args:
        params["message"] = f"Ты сказал: {args}"
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    return None

def cmd_device(self, plugin, args, params):
    """Показать информацию об устройстве"""
    simple_info = Mandre.Device.get_simple_info()
    params["message"] = f"📱 {simple_info}"
    return HookResult(strategy=HookStrategy.MODIFY, params=params)
```

Префикс команды настраивается в настройках MandreLib (по умолчанию `.`).

---

## 11. Локализация и TARGETlang ⭐ НОВОЕ

### Базовая локализация

```python
class MyPlugin(BasePlugin):
    TARGETlang = "ru"  # Укажите язык вашего плагина
    
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
    
    def some_function(self):
        # Используем переводы
        greeting = Mandre.t(self, "greeting", name="Алиса")
        BulletinHelper.show_info(greeting)  # Автоматически переведёт на язык пользователя
```

### Автоматический перевод настроек

При использовании `Mandre.Settings.render()` все строки автоматически переводятся:

```python
def create_settings(self):
    markup = """
    <screen text="Settings">
        <header text="Main Parameters"/>
        <switch key="enabled" text="Enable feature" subtext="Turn on functionality"/>
        <button text="Apply" icon="msg_check"/>
    </screen>
    """
    return Mandre.Settings.render(self, markup)
```

MandreLib автоматически:
1. Собирает все видимые строки из DSL
2. Переводит их через Pollinations.AI на язык пользователя
3. Кэширует переводы в `locales/<lang>.json`
4. Обновляет UI при готовности перевода

### Ручной запуск перевода

```python
# Для своих кастомных строк
Mandre.auto_translate_inline_strings(
    plugin_instance=self,
    strings=[
        "Моя кастомная строка 1",
        "Моя кастомная строка 2",
    ]
)
```

**Язык пользователя** определяется автоматически. Если `TARGETlang` плагина совпадает с языком пользователя — перевод не выполняется.

---

## 12. Превью чата в настройках (Chat Preview) ⭐ НОВОЕ

```python
SAMPLE_MESSAGES = [
    {"name": "Alice", "text": "Hello! This is a sample.", "avatar": "https://i.pravatar.cc/150?img=1"},
    {"name": "Bob",   "text": "Okay, showing preview in settings.", "avatar": "https://i.pravatar.cc/150?img=2"},
]

def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    # Добавить превью в настройки (2 строки!)
    Mandre.Settings.add_chat_preview(self, SAMPLE_MESSAGES)
```

### Включение/выключение превью

Добавьте тумблер **последним** в настройках:

```python
def create_settings(self):
    return Mandre.Settings.render(self, """
    <screen text="Main">
        <header text="Demo Settings"/>
        <switch key="feature_enabled" text="Enable Feature"/>
        
        <!-- Тумблер для превью ДОЛЖЕН быть последним -->
        <switch key="preview_enabled" text="Show chat preview" subtext="Displays messages below" icon="msg_settings" default="true"/>
    </screen>
    """)
```

**Особенности:**
- Автоперевод сообщений работает автоматически
- Длинные строки переносятся в пузыре
- Безопасный long-press (не крашится)
- Аватары скругляются и загружаются по URL
- Поддерживается до 10 сообщений

---

## 13. Кастомное меню автодополнения (MandreSuggestions) ⭐ НОВОЕ

Создайте свои меню автодополнения, как встроенные `@упоминания` и `/команды`.

### Базовая регистрация

```python
class ToxicReply(BasePlugin):
    TARGETlang = "ru"
    
    def on_plugin_load(self):
        trigger = "пошел нахуй"
        menu_dsl = """
        <item text="сам пошел" subtext="Зеркальный ответ ↩️" />
        <item text="извинись" subtext="Быкануть в ответ 🐂" />
        """
        Mandre.Suggestions.register(self, trigger, menu_dsl)
```

### Динамическое создание меню

```python
def create_suggestions_menu(self):
    # Можно вернуть строку DSL или сгенерировать динамически
    return """
    <item text="Option 1" subtext="Description 1" />
    <item text="Option 2" subtext="Description 2" />
    """

def on_plugin_load(self):
    Mandre.Suggestions.register(self, "@mybot", self.create_suggestions_menu)
```

### Вставка текста программно

```python
# Вставить текст в поле ввода
Mandre.Suggestions.trigger_input("@mybot ")
```

### Комплексный пример

```python
def on_plugin_load(self):
    # Регистрируем несколько триггеров
    Mandre.Suggestions.register(self, "#code", self.get_code_snippets)
    Mandre.Suggestions.register(self, "//tools", self.get_tools_menu)

def get_code_snippets(self):
    # Генерация на основе настроек
    snippets = self.get_setting("code_snippets", [])
    dsl = ""
    for snippet in snippets:
        dsl += f'<item text="{snippet["name"]}" subtext="{snippet["desc"]}" />\n'
    return dsl

def get_tools_menu(self):
    return """
    <item text="Generate ID" subtext="Создать уникальный ID" />
    <item text="Timestamp" subtext="Текущее время" />
    <item text="Random" subtext="Случайное число" />
    """
```

---

## 14. Мгновенный доступ к сообщениям (MandreMessages) ⭐ НОВОЕ

Получайте сообщения напрямую из SQLite без сетевых запросов — **мгновенно**.

```python
def read_chat_history(self, dialog_id: int, limit: int = 50):
    # Получаем сообщения из локальной базы
    messages = Mandre.Messages.get_local(dialog_id, limit)
    
    # Форматируем для показа
    result = []
    for msg in messages:
        text = msg.message or "[Медиа/Без текста]"
        sender = msg.from_id.user_id if hasattr(msg, 'from_id') else "Unknown"
        date = time.strftime('%H:%M', time.localtime(msg.date))
        result.append(f"{date} | {sender}: {text}")
    
    return "\n".join(result)

def on_menu_click(self, ctx):
    dialog_id = ctx.get("dialog_id")
    if dialog_id:
        history = self.read_chat_history(dialog_id, 20)
        Mandre.Share.share_text(history, "История чата")
```

**Важно:**
- Работает офлайн
- Возвращает список объектов `TLRPC.Message`
- Поддерживает все типы сообщений (текст, медиа, файлы)
- Быстрее API в 100+ раз

---

## 15. Работа с SQL (KV-хранилище)

### KV-операции

```python
# Создать таблицу
Mandre.sql_init_kv(self.id, "my_kv_table")

# Записать значение
Mandre.sql_kv_set(self.id, "user:12345", "premium", "my_kv_table")

# Прочитать значение
value = Mandre.sql_kv_get(self.id, "user:12345", "my_kv_table")
# "premium"

# Прочитать число
count = Mandre.sql_kv_get_int(self.id, "counter", 0, "my_kv_table")

# Удалить по префиксу
Mandre.sql_kv_delete_prefix(self.id, "user:", "my_kv_table")
```

### Синтетические каналы

```python
# Создать виртуальный канал (без запроса к серверу)
channel = Mandre.register_synthetic_channel(
    channel_id=777777777, 
    title="Моя Стена",
    megagroup=False, 
    broadcast=True
)

# Использовать в MessagesController
# Это позволяет работать с каналом как с реальным чатом
```

---

## 16. Deep linking (tg:// ссылки)

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Регистрируем обработчик
    Mandre.add_tg_alias("my_plugin/action", self.handle_deeplink)
    
    # Автоматическая ссылка на настройки
    Mandre.register_settings_alias(self)

def handle_deeplink(self, intent):
    uri_str = str(intent.getData())
    self.log(f"Открыта ссылка: {uri_str}")
    
    if "action=backup" in uri_str:
        self._create_backup()
    elif "action=restore" in uri_str:
        self._show_restore_dialog()

def on_plugin_unload(self):
    Mandre.remove_tg_alias("my_plugin/action")
```

### Сложные deep links

```python
def handle_complex_deeplink(self, intent):
    uri = intent.getData()
    path = uri.getPath()
    
    if path == "/dashboard":
        self._open_dashboard()
    elif path.startswith("/reminder/"):
        reminder_id = path.split("/")[-1]
        self._show_reminder(reminder_id)
    
    # Обработка query параметров
    if uri.getQueryParameter("auto_open") == "true":
        self._enable_auto_open()
```

---

## 17. Полный пример: Плагин мониторинга устройства

```python
__id__ = "device_monitor"
__name__ = "Device Monitor"
__version__ = "1.0"
__author__ = "@You"
__description__ = "Мониторинг состояния устройства в реальном времени"
__min_version__ = "11.9.0"

from base_plugin import BasePlugin, HookResult, HookStrategy
from mandre_lib import Mandre, MandreDevice, MandreNotification, MandreMessages
import time

class DeviceMonitorPlugin(BasePlugin):
    TARGETlang = "ru"
    
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # Команды
        Mandre.register_command(self, "status", self.cmd_status)
        Mandre.register_command(self, "battery", self.cmd_battery)
        
        # Планировщик задач
        Mandre.schedule_task(self, "monitor", 60, self.monitor_system)
        Mandre.schedule_task(self, "cleanup", 3600, self.cleanup_logs)
        
        self.log("Device Monitor загружен")

    def cmd_status(self, plugin, args, params):
        """Показать статус системы"""
        info = MandreDevice.get_device_info()
        battery = self._get_battery_level()
        
        status = f"""📊 Статус системы

💾 Память: {info.get('available_memory_mb', '?')} МБ свободно
🔋 Батарея: {battery or '?'}%
🌡️ CPU: {self._get_cpu_temp() or '?'}°C
📱 Android: {info.get('android_version', '?')}"""
        
        params["message"] = status
        return HookResult(strategy=HookStrategy.MODIFY, params=params)

    def monitor_system(self):
        info = MandreDevice.get_device_info()
        
        # Проверка температуры
        temp = self._get_cpu_temp()
        if temp and temp > 75:
            Mandre.Notification.show_simple(
                "⚠️ Перегрев",
                f"Температура CPU: {temp}°C",
                channel_id="thermal_warnings"
            )
        
        # Проверка памяти
        available = info.get("available_memory_mb", 0)
        if available < 200:
            Mandre.Notification.show_simple(
                "💾 Мало памяти",
                f"Доступно {available} МБ",
                channel_id="memory_warnings"
            )

    def _get_battery_level(self):
        # Реализация получения заряда батареи...
        pass

    def _get_cpu_temp(self):
        # Реализация получения температуры CPU...
        pass

    def create_settings(self):
        return Mandre.Settings.render(self, """
        <screen text="Monitor Settings">
            <header text="Device Monitor v1.0"/>
            
            <switch key="monitor_enabled" text="Enable monitoring" icon="msg_settings" default="true"/>
            <switch key="temp_alerts" text="Temperature alerts" icon="msg_warning" default="true"/>
            <switch key="memory_alerts" text="Memory alerts" icon="msg_memory" default="true"/>
            
            <divider text="Actions"/>
            <button text="Test notification" icon="msg_notifications" on_click="test_notify"/>
            <button text="View logs" icon="msg_log" on_click="view_logs"/>
            
            <switch key="preview_enabled" text="Show preview" default="true"/>
        </screen>
        """)
    
    def test_notify(self):
        Mandre.Notification.show_dialog(
            sender_name="Test",
            message="Test notification",
            avatar_url="https://i.pravatar.cc/150?img=1"
        )
```

---

## 18. Миграция с 1.6.6 на 1.7

### 🔴 ВАЖНО: Критические изменения

1. **Mandre.Pip теперь MandreInstall**
   - ❌ Старый: `Mandre.Pip.ensure_ready()`
   - ✅ Новый: `MandreInstall("check_status")` или `MandreInstall(["requests"])`

2. **Локализация переработана**
   - ❌ Старый: `Mandre.t(self, "key")` + ручной `register_localizations`
   - ✅ Новый: Просто укажите `TARGETlang = "ru"` и используйте `Mandre.t()`
   - ✅ Strings в `Mandre.Settings.render()` или `<item text="...">` автоматически переводятся

3. **Настройки через DSL**
   - ❌ Старый: `create_settings()` возвращает список объектов
   - ✅ Новый: Рекомендуется использовать `Mandre.Settings.render()` с HTML-like DSL

4. **Картинки в настройках удалены**
   - ❌ Старый: `<image>` и `<media>` теги
   - ✅ Новый: Используйте `Mandre.Settings.add_chat_preview()` для визуализации

5. **Новый PIP менеджер**
   - Установка пакетов теперь проще: `MandreInstall("package")`
   - Поддержка wheel файлов и ZIP архивов
   - Автоматическое разрешение зависимостей

### 🟡 Рекомендации по обновлению

1. Добавьте `TARGETlang = "ru"` (или ваш язык) в класс плагина
2. Замените `Mandre.Pip.ensure_ready()` на `MandreInstall("check_status")`
3. Перепишите сложные `create_settings()` на DSL
4. Удалите все вызовы `register_localizations` — они больше не нужны
5. Замените кастомные UI на `MandreUI.show_bottom_sheet()` где возможно
6. Для отправки изображений используйте `MandreSend.png()`

---

## 19. Распространенные ошибки и их решения

### ❌ Ошибка: "MandreLib не найдена"
```python
# Убедитесь, что плагин MandreLib установлен и включен
# ID плагина: mandre_lib
```

### ❌ Ошибка: "Пакет не устанавливается"
```python
# Используйте новый синтаксис:
MandreInstall("requests")

# Не забывайте проверять статус:
if not MandreInstall("check_status"):
    BulletinHelper.show_error("PIP не готов")
```

### ❌ Ошибка: "Автоматический перевод не работает"
```python
# 1. Добавьте TARGETlang в класс
class MyPlugin(BasePlugin):
    TARGETlang = "ru"
    
# 2. Убедитесь, что есть интернет (первый перевод)
# 3. Проверьте кэш: plugins/mandre_lib_data/<plugin_id>/locales/
```

### ❌ Ошибка: "Settings не рендерятся"
```python
# Используйте двойные кавычки в DSL!
# Правильно: <button text="Click me"/>
# Неправильно: <button text='Click me'/>
```

### ❌ Ошибка: "BottomSheet не открывается"
```python
# Проверьте структуру DSL — все теги должны быть закрыты
# <sheet> должен содержать <actions> и <content> корректно
```

### ✅ Правильная структура плагина 1.7

```python
__id__ = "my_plugin"
__name__ = "My Plugin"
__version__ = "1.0"
__min_version__ = "11.9.0"

from base_plugin import BasePlugin
from mandre_lib import Mandre

class MyPlugin(BasePlugin):
    TARGETlang = "ru"  # Обязательно!
    
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        # ... ваш код ...
```

---

## 20. Полный справочник API (версия 1.7)

### Mandre (главный класс)

| Метод | Описание | Возвращает |
|-------|----------|------------|
| `use_persistent_storage(plugin)` | Активация автосохранения настроек | None |
| `schedule_task(plugin, name, interval, callback)` | Запуск повторяющейся задачи | None |
| `cancel_task(plugin, name)` | Отмена задачи | None |
| `register_command(plugin, name, callback)` | Регистрация команды | None |
| `handle_outgoing_command(params)` | Обработка команд в хуке | HookResult \| None |
| `add_tg_alias(path, callback)` | Регистрация tg:// ссылки | None |
| `remove_tg_alias(path)` | Удаление ссылки | None |
| `register_settings_alias(plugin)` | Автоссылка на настройки | None |
| `apply_and_refresh_settings(plugin)` | Принудительное обновление UI | None |
| `t(plugin, key, **kwargs)` | Локализация строки | str |
| `auto_translate_inline_strings(plugin, strings)` | Автоперевод списка строк | None |
| `register_synthetic_channel(id, title, megagroup, broadcast)` | Создание виртуального чата | TLRPC.Chat \| None |
| `sql_get_database()` | Доступ к SQLite | SQLiteDatabase |
| `sql_init_kv(plugin_id, table_name)` | Создание KV таблицы | None |
| `sql_kv_set(plugin_id, key, value, table_name)` | Запись в KV | None |
| `sql_kv_get(plugin_id, key, table_name)` | Чтение из KV | str \| None |
| `sql_kv_get_int(plugin_id, key, default, table_name)` | Чтение числа из KV | int |
| `sql_kv_delete_prefix(plugin_id, prefix, table_name)` | Удаление по префиксу | None |

### MandreData

| Метод | Описание | Возвращает |
|-------|----------|------------|
| `write_persistent_json(plugin_id, filename, data)` | Запись JSON | None |
| `read_persistent_json(plugin_id, filename, default)` | Чтение JSON | Any |
| `get_persistent_path(plugin_id, filename)` | Путь к файлу | str |
| `list_persistent_plugins()` | Список плагинов с данными | list[str] |
| `list_files_for_plugin(plugin_id)` | Список файлов плагина | list[str] |
| `delete_persistent_plugin_data(plugin_id)` | Удаление данных плагина | bool |

### MandreUI

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `show(title, items, on_select, message, cancel_text)` | Диалог выбора | title, items, callback, message, cancel_text |
| `select_chat(title, on_select, search_hint, cancel_text)` | Селектор чатов | title, callback, search_hint, cancel_text |
| `ripple(intensity, vibrate)` | Эффект волны | intensity=2.0, vibrate=True |
| `setup_settings_bottom_bar(plugin, items, ...)` | Нижняя панель в настройках | plugin, items, active_index_key, colors... |
| `update_bottom_bar(plugin_id, active_index)` | Обновить активную вкладку | plugin_id, active_index |
| `show_bottom_sheet(plugin, dsl, callbacks)` | BottomSheet с DSL | plugin, dsl_string, callbacks_dict |

### MandreSettings

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `render(plugin, spec)` | Рендер DSL в настройки | plugin, dsl_string \| dict |
| `add_chat_preview(plugin, messages)` | Добавить превью чата | plugin, list[dict] |

### MandreInstall (НОВОЕ)

| Вызов | Описание | Возвращает |
|-------|----------|------------|
| `MandreInstall("package")` | Установить пакет/URL/список | None |
| `MandreInstall("check_status")` | Проверить готовность PIP | bool |

### MandrePip (низкоуровневый)

| Метод | Описание | Возвращает |
|-------|----------|------------|
| `ensure_ready(silent=False)` | Инициализация PIP | bool |
| `pip(args)` | Выполнить pip команду | tuple(code, out, err) |
| `install(spec)` | Установить пакет | tuple |
| `site_dir()` | Путь к site-packages | str |
| `import_module(mod)` | Импортировать модуль | module |

### MandreShare

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `share_text(text, title)` | Поделиться текстом | text, title="Поделиться" |
| `share_file(file_path, title, mime_type)` | Поделиться файлом | file_path, title, mime_type |

### MandreSend (НОВОЕ)

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `png(path, caption)` | Отправить PNG в чат | path, caption=None |

### MandreDevice

| Метод | Описание | Возвращает |
|-------|----------|------------|
| `get_device_info()` | Полная информация об устройстве | dict |
| `get_simple_info()` | Краткая строка | str |

### MandreNotification

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `show_simple(title, text, channel_id)` | Простое уведомление | title, text, channel_id |
| `show_dialog(sender_name, message, avatar_url, channel_id)` | Диалог-стиль | sender, message, avatar, channel_id |

### MandreTTS

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `speak(text)` | Озвучить текст | text: str |

### MandreAuth

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `request(on_success, on_failure, title, description)` | Запросить аутентификацию | callbacks, title, description |

### MandreWeb

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `render_html_to_png(html, on_result, width, height, ...)` | HTML в PNG | html, callback, width=1024, height=768, ... |

### MandreMessages (НОВОЕ)

| Метод | Описание | Аргументы | Возвращает |
|-------|----------|-----------|------------|
| `get_local(dialog_id, limit)` | Сообщения из SQLite | dialog_id, limit=100 | list[TLRPC.Message] |

### MandreSuggestions (НОВОЕ)

| Метод | Описание | Аргументы |
|-------|----------|-----------|
| `register(plugin, trigger, content)` | Регистрация меню | plugin, trigger, dsl \| callable |
| `trigger_input(text)` | Вставить текст | text: str |

---

## 21. Практические шаблоны

### Шаблон 1: Плагин с BottomBar и превью

```python
__id__ = "demo_full"
__name__ = "Full Demo"
__version__ = "1.0"
TARGETlang = "ru"

from base_plugin import BasePlugin
from mandre_lib import Mandre, MandreUI
from android.graphics import Color

class FullDemoPlugin(BasePlugin):
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # BottomBar
        items = [
            {"text": "Tab 1", "icon": "msg_settings", "on_click": lambda: self._switch_tab(0)},
            {"text": "Tab 2", "icon": "msg_storage", "on_click": lambda: self._switch_tab(1)},
        ]
        Mandre.UI.setup_settings_bottom_bar(self, items, "current_tab", 
            bg_color=Color.argb(210, 50, 50, 55),
            active_color=Color.WHITE,
            inactive_color=Color.rgb(140, 140, 140)
        )
        
        # Chat Preview
        messages = [
            {"name": "Bot", "text": "Welcome!", "avatar": "https://i.pravatar.cc/150?img=3"},
            {"name": "User", "text": "Thanks!", "avatar": "https://i.pravatar.cc/150?img=4"},
        ]
        Mandre.Settings.add_chat_preview(self, messages)
    
    def _switch_tab(self, index):
        self.set_setting("current_tab", index)
        Mandre.apply_and_refresh_settings(self)
    
    def create_settings(self):
        return Mandre.Settings.render(self, f"""
        <screen>
            <header text="Tab {self.get_setting('current_tab', 0)}"/>
            <switch key="feature" text="Feature" icon="msg_settings"/>
        </screen>
        """)
```

### Шаблон 2: Плагин с Suggestions и Commands

```python
__id__ = "code_helper"
__name__ = "Code Helper"
__version__ = "1.0"
TARGETlang = "en"

from base_plugin import BasePlugin, HookResult
from mandre_lib import Mandre

class CodeHelperPlugin(BasePlugin):
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # Регистрация команд
        Mandre.register_command(self, "code", self.cmd_code)
        
        # Регистрация suggestions
        Mandre.Suggestions.register(self, "#py", self.get_python_snippets)
        Mandre.Suggestions.register(self, "#js", self.get_js_snippets)
        
        self.add_on_send_message_hook()
    
    def on_send_message_hook(self, params):
        return Mandre.handle_outgoing_command(params) or HookResult()
    
    def cmd_code(self, plugin, args, params):
        params["message"] = f"```\n{args}\n```"
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    def get_python_snippets(self):
        return """
        <item text="for loop" subtext="for i in range(10):" />
        <item text="function" subtext="def func():" />
        <item text="class" subtext="class MyClass:" />
        """
    
    def get_js_snippets(self):
        return """
        <item text="function" subtext="function test() {}" />
        <item text="arrow" subtext="const test = () => {}" />
        """
```

---

## 22. Заключение

MandreLib 1.7 — это мощнейший инструментарий для создания профессиональных плагинов exteraGram:

✅ **MandreInstall** — однострочная установка пакетов  
✅ **MandreSettings DSL** — современные настройки без boilerplate  
✅ **Chat Preview** — визуализация прямо в настройках  
✅ **MandreSuggestions** — кастомные меню автодополнения  
✅ **MandreMessages** — мгновенный доступ к истории сообщений  
✅ **MandreSend** — прямая отправка медиа в чат  
✅ **TARGETlang** — нативная многоязычность  
✅ **BottomSheet** — современный UI  
✅ **Все классы 1.6.6** + улучшенная стабильность  

**Передавайте эту документацию AI-ассистентам для генерации кода. Удачи в разработке! 🚀**

---

## 📚 Дополнительные ресурсы
- **Чат поддержки**: `https://t.me/mandrelib`
- **API Pollinations**: `https://pollinations.ai/`

*Последнее обновление: ноябрь 2025*
