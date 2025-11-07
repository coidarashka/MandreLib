# MandreLib - Полная документация для разработчиков

*Документация обновлена для версии 1.6.6*

---

## Что такое MandreLib?

MandreLib — это универсальная библиотека для разработки плагинов в exteraGram. Она предоставляет готовые решения для самых частых задач: сохранение данных, планирование задач, создание UI-элементов, текст-в-речь, аутентификация, отправка файлов, получение информации об устройстве, уведомления, PIP-менеджер для Python пакетов, веб-рендеринг и многое другое. Вместо того чтобы писать всё с нуля, ты подключаешь MandreLib и используешь её функции.

Главная фишка — она экономит кучу времени. Все сложные вещи уже реализованы и протестированы.

---

## Базовая интеграция

Чтобы использовать MandreLib, сначала нужно её импортировать и активировать:

```python
from mandre_lib import Mandre, MandreData, MandreUI, MandreTTS, MandreAuth, MandreShare, MandreDevice, MandreNotification, MandrePip, MandreWeb, MandreSend
from base_plugin import BasePlugin

class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        # Активируем персистентное хранилище (сохранение данных)
        Mandre.use_persistent_storage(self)
        
        # Опционально: убедимся что PIP готов (для Python пакетов)
        Mandre.Pip.ensure_ready()
        
        self.log("Плагин загружен, MandreLib активирована")
```

Вот и всё. Теперь ты можешь использовать все возможности библиотеки.

---

## 0. SQL KV-хелперы и синтетический канал ⭐

### SQL KV (таблица по умолчанию `mandre_kv`)
- `Mandre.sql_get_database()` — получить объект БД `MessagesStorage`.
- `Mandre.sql_init_kv(plugin_id, table_name="mandre_kv")` — создать KV-таблицу (если нет).
- `Mandre.sql_kv_set(plugin_id, key, value, table_name="mandre_kv")` — записать значение.
- `Mandre.sql_kv_get(plugin_id, key, table_name="mandre_kv") -> Optional[str]` — прочитать строку.
- `Mandre.sql_kv_get_int(plugin_id, key, default=0, table_name="mandre_kv") -> int` — прочитать число.
- `Mandre.sql_kv_delete_prefix(plugin_id, prefix, table_name="mandre_kv")` — удалить все ключи с префиксом.

Пример:
```python
Mandre.sql_init_kv(self.id, "wallfeed_kv")
Mandre.sql_kv_set(self.id, "meta:last_sync", int(time.time()), "wallfeed_kv")
last_sync = Mandre.sql_kv_get_int(self.id, "meta:last_sync", 0, "wallfeed_kv")
```

### Регистрация синтетического канала
- `Mandre.register_synthetic_channel(channel_id, title, megagroup=False, broadcast=True)`  
Создаёт/обновляет чат в `MessagesController` без сетевого запроса.

Пример:
```python
Mandre.register_synthetic_channel(777777777, "Стена", megagroup=False, broadcast=True)
```

---

## 1. Персистентное хранилище данных (MandreData)

Это самая полезная фишка. Обычно данные плагина хранятся в памяти и теряются при перезагрузке. MandreLib сохраняет всё на диск автоматически.

### Как это работает

Когда ты активируешь `Mandre.use_persistent_storage(self)`, все вызовы `self.set_setting(key, value)` автоматически сохраняются в файл. При следующей загрузке плагина данные восстанавливаются.

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Восстановится из сохранённого файла
    count = self.get_setting("counter", 0)
    self.set_setting("counter", count + 1)
    self.log(f"Плагин загружался {count + 1} раз")
```

### Работа с JSON-файлами напрямую

Если тебе нужно больше контроля, используй `MandreData` напрямую:

```python
# Написать JSON
my_data = {
    "users": [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}],
    "settings": {"theme": "dark"}
}
MandreData.write_persistent_json(self.id, "database.json", my_data)

# Прочитать JSON
loaded_data = MandreData.read_persistent_json(self.id, "database.json", default={})
self.log(f"Загруженные данные: {loaded_data}")

# Список всех файлов для плагина
files = MandreData.list_files_for_plugin(self.id)
self.log(f"Файлы: {files}")
```

Всё автоматически потокобезопасно — можно писать из фоновых потоков без проблем.

### Импорт и экспорт данных

MandreLib поддерживает полноценный импорт/экспорт данных плагинов:

```python
# Экспорт данных в ZIP-архив
def export_plugin_data(self):
    downloads_dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    zip_filename = f"mandrelib_data_{self.id}_{timestamp}.zip"
    zip_path = os.path.join(downloads_dir.getAbsolutePath(), zip_filename)

    with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
        for filename in MandreData.list_files_for_plugin(self.id):
            file_path = MandreData.get_persistent_path(self.id, filename)
            zipf.write(file_path, arcname=filename)
    
    BulletinHelper.show_success(f"Данные экспортированы в Downloads/{zip_filename}")

# Импорт данных из ZIP-архива
def import_plugin_data(self, zip_path):
    MandreData.delete_persistent_plugin_data(self.id)
    target_dir = File(MandreData._get_base_data_dir(), self.id)
    if not target_dir.exists(): target_dir.mkdirs()
    
    with zipfile.ZipFile(zip_path, 'r') as zipf:
        zipf.extractall(target_dir.getAbsolutePath())
    
    BulletinHelper.show_success("Данные импортированы!")
```

---

## 2. UI-компоненты (MandreUI)

MandreLib предоставляет готовые методы для показа диалогов и уведомлений.

### Локализация и авто‑перевод настроек ⭐ НОВОЕ

MandreLib теперь умеет автоматически переводить строки интерфейса настроек и подставлять переводы через единый вызов `Mandre.t(...)`.

- `Mandre.t(text_or_key)` — возвращает перевод для заданной строки/ключа, если он уже доступен, иначе возвращает исходный текст.
- Авто‑сбор строк: `MandreSettings.render` автоматически собирает видимые строки (заголовки, подписи, тексты кнопок, подсказки и т.п.) и запускает фоновый перевод на выбранный язык.
- Хранилище переводов: JSON‑файлы сохраняются по пути `.../plugins/mandre_lib_data/<plugin_id>/locales/<lang>.json` (каталог создаётся автоматически).
- Кэш и устойчивость: уже переведённые строки не запрашиваются повторно. Ваши ручные правки в JSON сохранятся.
- Как использовать: просто оборачивайте все видимые тексты в `Mandre.t(...)`.

Пример использования в настройках плагина:

```python
from ui.settings import Header, Text, Input, Switch

def create_settings(self):
    return [
        Header(text=Mandre.t("Основные параметры")),
        Text(
            text=Mandre.t("Это пример настроек."),
            subtext=Mandre.t("Строки будут переведены фоном")
        ),
        Input(
            key="username",
            text=Mandre.t("Имя пользователя"),
            subtext=Mandre.t("Введите имя"),
            default=""
        ),
        Switch(
            key="feature_on",
            text=Mandre.t("Включить функцию"),
            default=False
        ),
    ]
```

Ручной вызов авто‑перевода вне экрана настроек (не обязателен для `MandreSettings`, но полезен в своём UI):

```python
Mandre.auto_translate_inline_strings(
    plugin_id=self.id,
    strings=[
        "Основные параметры",
        "Имя пользователя",
        "Сохранить",
    ],
    lang=self.get_setting("language", "en")  # целевой язык
)
```

Дополнительно: ключевая локализация

Для фиксированных ключей (например, системные сообщения/строки, которые вы хотите иметь в словаре локалей) используйте регистрацию локализаций:

```python
Mandre.register_localizations(
    plugin_id=self.id,
    base_texts={
        "settings.header": "Основные параметры",
        "settings.username": "Имя пользователя",
        "settings.save": "Сохранить",
    }
)
# Фоновые переводы будут подготовлены по целевому языку,
# а Mandre.t("settings.save") вернёт перевод при наличии
```

Поддерживаемые элементы настроек и локализация:
- `Header.text`
- `Divider.text`
- `Text.text`, `Text.subtext`
- `Button.text`
- `Input.text`, `Input.subtext`, `Input.placeholder` (если используется)
- `Switch.text`
- `Selector.text`, `Selector.items` (отображаемые подписи)

Важно (миграция):
- Элементы настроек `image` и `media` больше не поддерживаются и не рендерятся.
- Внутренний метод `Mandre.Settings._open_media` удалён. Если ваши плагины его вызывали, пожалуйста, удалите соответствующий код или замените на новый UX.
- Сетевое подключение обязательно для фонового перевода. Переводы кэшируются локально.

### Простой диалог с выбором

```python
def show_options(self):
    MandreUI.show(
        title="Выберите действие",
        items=["Первое", "Второе", "Третье"],
        on_select=lambda index, text: self.log(f"Выбран пункт {index}: {text}"),
        message="Какой вариант тебе нравится?",
        cancel_text="Отмена"
    )
```

### Ripple эффект (визуальная обратная связь)

```python
# Показать волну в центре экрана
MandreUI.ripple(intensity=2.0, vibrate=True)
```

### Селектор чатов

Для выбора чата или пользователя используй встроенный селектор:

```python
def select_chat_handler(chat_info):
    self.log(f"Выбран чат: {chat_info['title']} (ID: {chat_info['id']})")
    # chat_info содержит: 'title', 'id', 'obj' (объект чата или пользователя)

MandreUI.select_chat(
    title="Выберите чат",
    on_select=select_chat_handler,
    search_hint="Поиск чата...",
    cancel_text="Отмена"
)
```

### Навигационная панель в настройках (BottomBar)

Добавить вкладки/кнопки в нижней части экрана настроек плагина.

```python
from android.graphics import Color
from ui.settings import Header

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
    elif tab == 1:
        return [Header(text="Управление данными")]
```

---

## 3. PIP менеджер (MandrePip) ⭐ НОВОЕ

MandreLib теперь включает полноценный PIP-менеджер для установки Python пакетов в Android окружении.

### Быстрая установка пакетов

```python
# Установить один пакет
Mandre.Pip.install("requests")

# Установить несколько пакетов одновременно
Mandrelib(["pyrogram", "asyncio", "aiohttp"])

# Убедиться что PIP готов
Mandre.Pip.ensure_ready()
```

### Работа с PIP напрямую

```python
# Установка с дополнительными параметрами
code, out, err = Mandre.Pip.pip([
    "install", 
    "--upgrade", 
    "--no-warn-conflicts",
    "colorama==0.4.6"
])

if code == 0:
    BulletinHelper.show_success("Пакет установлен!")
else:
    BulletinHelper.show_error(f"Ошибка: {err}")

# Проверить установленные пакеты
code, out, err = Mandre.Pip.pip(["list"])
self.log(f"Установленные пакеты:\n{out}")
```

### Импорт установленных модулей

```python
# Установка и импорт в одном флаконе
try:
    import requests
except ImportError:
    Mandre.Pip.install("requests")
    requests = Mandre.Pip.import_module("requests")

# Использование установленного пакета
response = requests.get("https://api.github.com/users/octocat")
if response.status_code == 200:
    data = response.json()
    self.log(f"Пользователь: {data['name']}")
```

### Кастомные настройки PIP

Настройки PIP можно конфигурировать через настройки MandreLib:

```python
def create_settings(self):
    return [
        # ... другие настройки ...
        
        Input(
            key="pip_index_url", 
            text="Индекс пакетов (опционально)",
            subtext="Кастомный PyPI индекс",
            default="",
            icon="msg_edit_solar"
        ),
        
        Input(
            key="pip_proxy", 
            text="HTTP прокси (опционально)",
            subtext="Формат: http://user:pass@host:port",
            default="",
            icon="msg_edit_solar"
        ),
    ]
```

---

## 4. Веб-рендеринг (MandreWeb) ⭐ НОВОЕ

MandreWeb позволяет рендерить HTML в PNG изображения с помощью WebView.

### Рендеринг HTML в изображение

```python
def render_html_to_image(self):
    html_content = """
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <style>
            body { font-family: Arial, sans-serif; padding: 20px; background: #f0f0f0; }
            .card { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
            h1 { color: #333; }
        </style>
    </head>
    <body>
        <div class="card">
            <h1>📊 Отчёт плагина</h1>
            <p>Генерация отчёта успешно завершена!</p>
            <p>Время: <script>document.write(new Date().toLocaleString('ru-RU'))</script></p>
        </div>
    </body>
    </html>
    """
    
    def on_result(success: bool, result_path: str):
        if success:
            self.log(f"Изображение сохранено: {result_path}")
            # Отправляем изображение в чат
            MandreSend.png(result_path, "📊 Сгенерированный отчёт")
        else:
            BulletinHelper.show_error(f"Ошибка рендеринга: {result_path}")
    
    MandreWeb.render_html_to_png(
        html=html_content,
        on_result=on_result,
        width=1024,
        height=768,
        bg_color=(240, 240, 240)
    )
```

### Создание отчётов с динамическими данными

```python
def generate_device_report(self):
    device_info = MandreDevice.get_device_info()
    
    html = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <style>
            body {{ font-family: Arial, sans-serif; margin: 20px; background: #1a1a1a; color: white; }}
            .report {{ background: #2d2d2d; padding: 20px; border-radius: 10px; }}
            h1 {{ color: #4CAF50; text-align: center; }}
            .info {{ margin: 10px 0; padding: 10px; background: #3d3d3d; border-radius: 5px; }}
            .label {{ font-weight: bold; color: #4CAF50; }}
        </style>
    </head>
    <body>
        <div class="report">
            <h1>📱 Отчёт об устройстве</h1>
            <div class="info">
                <span class="label">Устройство:</span> {device_info.get('manufacturer', 'Unknown')} {device_info.get('model', 'Unknown')}
            </div>
            <div class="info">
                <span class="label">Android:</span> {device_info.get('android_version', 'Unknown')} (API {device_info.get('api_level', 'Unknown')})
            </div>
            <div class="info">
                <span class="label">Память:</span> {device_info.get('total_memory_mb', 'Unknown')} МБ
            </div>
            <div class="info">
                <span class="label">Экран:</span> {device_info.get('screen_width', 'Unknown')}x{device_info.get('screen_height', 'Unknown')}
            </div>
        </div>
    </body>
    </html>
    """
    
    MandreWeb.render_html_to_png(html, self._on_report_ready, file_prefix="device_report_")

def _on_report_ready(self, success: bool, path: str):
    if success:
        MandreSend.png(path, "📱 Отчёт об устройстве")
        BulletinHelper.show_success("Отчёт готов!")
    else:
        BulletinHelper.show_error("Не удалось создать отчёт")
```

---

## 5. Отправка файлов и текста (MandreShare) ⭐ ОБНОВЛЕНО

MandreShare позволяет легко отправлять текст или файлы через системный диалог "Поделиться" Android.

### Отправка текста

```python
def share_greeting(self):
    Mandre.Share.share_text(
        text="Привет! Это сообщение от моего плагина!",
        title="Поделиться приветствием"
    )
```

### Отправка файлов

```python
def share_document(self):
    # Путь к файлу (может быть временным файлом)
    file_path = "/path/to/document.pdf"
    
    Mandre.Share.share_file(
        file_path=file_path,
        title="Поделиться документом",
        mime_type="application/pdf"  # Опционально, определяется автоматически
    )
```

**Автоматическое определение MIME-типов:**
Библиотека автоматически определяет тип файла по расширению для изображений, видео, аудио, документов, архивов и веб-файлов.

### Создание и отправка временных файлов

```python
def create_and_share_text_file(self):
    import tempfile
    import time
    
    # Создаём временный файл
    with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False, encoding='utf-8') as f:
        f.write("Это содержимое временного файла\n")
        f.write(f"Создан: {time.strftime('%Y-%m-%d %H:%M:%S')}\n")
        temp_file_path = f.name
    
    # Отправляем файл
    Mandre.Share.share_file(temp_file_path, "Отправить текстовый файл")
    # Файл автоматически удалится через 5 минут
```

### Прямая отправка в чат (MandreSend) ⭐ НОВОЕ

Для отправки изображений прямо в текущий чат:

```python
def send_image_directly(self, image_path: str):
    # Отправить PNG изображение напрямую в текущий чат
    MandreSend.png(image_path, "📸 Изображение из плагина")
    
def create_and_send_chart(self):
    # Создаём HTML с графиком
    html = """
    <canvas id="myChart" width="400" height="200"></canvas>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script>
        const ctx = document.getElementById('myChart').getContext('2d');
        new Chart(ctx, {
            type: 'line',
            data: {
                labels: ['Янв', 'Фев', 'Мар', 'Апр', 'Май'],
                datasets: [{
                    label: 'Продажи',
                    data: [65, 59, 80, 81, 56],
                    borderColor: 'rgb(75, 192, 192)',
                    tension: 0.1
                }]
            }
        });
    </script>
    """
    
    def on_ready(success, path):
        if success:
            MandreSend.png(path, "📈 График продаж")
    
    MandreWeb.render_html_to_png(html, on_ready)
```

---

## 6. Информация об устройстве (MandreDevice) ⭐ НОВОЕ

MandreDevice предоставляет подробную информацию о текущем устройстве.

### Получение полной информации

```python
def get_device_info(self):
    device_info = Mandre.Device.get_device_info()
    
    if "error" in device_info:
        self.log(f"Ошибка: {device_info['error']}")
        return
    
    self.log(f"Производитель: {device_info['manufacturer']}")
    self.log(f"Модель: {device_info['model']}")
    self.log(f"Android: {device_info['android_version']} (API {device_info['api_level']})")
    self.log(f"Всего памяти: {device_info['total_memory_mb']} МБ")
    self.log(f"Root: {device_info['is_rooted']}")
    self.log(f"Эмулятор: {device_info['is_emulator']}")
```

### Краткая информация

```python
def show_simple_device_info(self):
    simple_info = Mandre.Device.get_simple_info()
    # Вернёт строку типа: "Samsung Galaxy S21 (Android 13, API 33)"
    BulletinHelper.show_info(simple_info)
```

### Создание подробного отчёта

```python
def create_device_report(self):
    info = MandreDevice.get_device_info()
    if "error" in info:
        BulletinHelper.show_error("Не удалось получить информацию об устройстве")
        return
    
    report = f"""
📱 ПОДРОБНАЯ ИНФОРМАЦИЯ ОБ УСТРОЙСТВЕ

🏭 Производитель: {info.get('manufacturer', 'Unknown')}
📱 Модель: {info.get('model', 'Unknown')}
🏷️ Бренд: {info.get('brand', 'Unknown')}
🔧 Устройство: {info.get('device', 'Unknown')}
🖥️ Дисплей: {info.get('screen_width', '?')}x{info.get('screen_height', '?')}
   Плотность: {info.get('screen_density', '?')}
💾 Память: {info.get('total_memory_mb', '?')} МБ общая
   Доступно: {info.get('available_memory_mb', '?')} МБ

🤖 ANDROID
Версия: {info.get('android_version', 'Unknown')}
API: {info.get('api_level', 'Unknown')}
Кодовое имя: {info.get('codename', 'Unknown')}
Сборка: {info.get('build_id', 'Unknown')}

📶 СЕТЬ
Оператор: {info.get('network_operator_name', 'Unknown')}
Тип сети: {info.get('phone_type', 'Unknown')}
SIM оператор: {info.get('sim_operator_name', 'Unknown')}

🔒 БЕЗОПАСНОСТЬ
Root: {'✅ Есть' if info.get('is_rooted') else '❌ Нет'}
Эмулятор: {'⚠️ Да' if info.get('is_emulator') else '✅ Нет'}

🌍 РЕГИОН
Язык: {info.get('locale', 'Unknown')}
Часовой пояс: {info.get('timezone', 'Unknown')}
"""
    
    Mandre.Share.share_text(report, "📱 Отчёт об устройстве")
```

---

## 7. Системные уведомления (MandreNotification) ⭐ НОВОЕ

MandreNotification позволяет показывать системные уведомления Android (не путать с Bulletin — встроенными уведомлениями Telegram).

### Простое уведомление

```python
def show_simple_notification(self):
    Mandre.Notification.show_simple(
        title="MandreLib Demo",
        text="Это простое уведомление от плагина! 🚀",
        channel_id="my_plugin_notifications"  # Опционально
    )
```

### Уведомление в стиле диалога (с аватаром)

```python
def show_dialog_notification(self):
    Mandre.Notification.show_dialog(
        sender_name="Мой Плагин",
        message="Привет! Это уведомление в стиле диалога с аватаром! 🎉",
        avatar_url="https://example.com/avatar.png",  # URL аватара
        channel_id="my_plugin_dialogs"  # Опционально
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
        # Дополнительно озвучиваем напоминание
        MandreTTS.speak(f"Напоминание: {text}")

    # Запускаем задачу через планировщик
    task_name = f"reminder_{int(time.time())}"
    Mandre.schedule_task(self, task_name, delay_seconds, send_reminder)
    BulletinHelper.show_success(f"Напоминание установлено на {delay_seconds} сек")
```

**Особенности:**
- Автоматическое создание каналов уведомлений (Android 8.0+).
- Аватары автоматически скругляются и загружаются по URL.
- Стилизация под Telegram.
- Поддержка уведомлений-диалогов в стиле мессенджера.

---

## 8. Текст-в-речь (MandreTTS)

Озвучивание текста с помощью системного TTS устройства.

```python
def speak_hello(self):
    MandreTTS.speak("Привет, мир!")

def on_send_message_hook(self, params):
    message = params.get("message", "")
    if message.startswith(".say "):
        text = message[5:]
        MandreTTS.speak(text)
        # Отменяем отправку самого сообщения-команды
        return HookResult(strategy=HookStrategy.CANCEL)
    return HookResult()
```

### Озвучивание с проверкой доступности

```python
def safe_speak(self, text: str):
    try:
        if not text.strip():
            return
        
        # Проверяем доступность TTS
        if not hasattr(_TTS_STATE, 'init_ok') or not _TTS_STATE.init_ok:
            BulletinHelper.show_info("Подготавливаю синтезатор речи...")
            # TTS будет инициализирован автоматически
        
        MandreTTS.speak(text)
        
    except Exception as e:
        self.log(f"Ошибка TTS: {e}")
        BulletinHelper.show_error("Не удалось озвучить текст")
```

На что стоит обратить внимание:
- Первый вызов может быть медленным (инициализация).
- TTS использует системный синтезатор речи (обычно Google TTS).
- Язык устанавливается автоматически на основе локали устройства.

---

## 9. Аутентификация (MandreAuth)

Запрос подтверждения личности через экран блокировки устройства.

```python
def request_auth(self):
    def on_auth_success():
        self.log("Пользователь подтвердил личность!")
        BulletinHelper.show_success("Доступ разрешён")
    
    def on_auth_failure():
        self.log("Аутентификация не удалась")
        BulletinHelper.show_error("Доступ запрещён")
    
    MandreAuth.request(
        on_success=on_auth_success,
        on_failure=on_auth_failure,
        title="Требуется подтверждение",
        description="Пожалуйста, подтвердите свою личность для доступа к настройкам безопасности"
    )
```

### Использование для защиты критических операций

```python
def delete_all_data(self):
    MandreAuth.request(
        on_success=self._confirm_delete,
        on_failure=lambda: BulletinHelper.show_info("Удаление отменено"),
        title="Подтверждение удаления",
        description="Введите пароль/код разблокировки для подтверждения"
    )

def _confirm_delete(self):
    MandreData.delete_persistent_plugin_data(self.id)
    BulletinHelper.show_success("Все данные удалены!")
```

Это полезно для защиты чувствительных операций. Библиотека сама обработает все технические детали с системой Android.

---

## 10. Планировщик задач (Scheduler)

Выполнение функции по расписанию (с определённым интервалом).

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
    # Эта функция будет вызвана каждые 30 секунд

def on_plugin_unload(self):
    # Отменить задачу при выгрузке плагина
    Mandre.cancel_task(self, "auto_check")
```

### Продвинутое использование планировщика

```python
def setup_monitoring(self):
    # Задача для мониторинга системы
    Mandre.schedule_task(self, "system_monitor", 60, self.monitor_system)
    
    # Задача для очистки временных файлов
    Mandre.schedule_task(self, "cleanup_temp", 300, self.cleanup_temp_files)

def monitor_system(self):
    try:
        # Проверяем состояние устройства
        device_info = MandreDevice.get_device_info()
        
        if device_info.get("available_memory_mb", 0) < 100:
            # Мало памяти - показываем уведомление
            Mandre.Notification.show_simple(
                "Предупреждение",
                "Мало свободной памяти. Рекомендуется очистить кэш.",
                channel_id="memory_warnings"
            )
        
        # Логируем состояние
        self.log(f"Мониторинг: доступно {device_info.get('available_memory_mb', '?')} МБ памяти")
        
    except Exception as e:
        self.log(f"Ошибка мониторинга: {e}")

def cleanup_temp_files(self):
    try:
        # Очищаем временные файлы старше часа
        import os
        import time
        
        temp_dir = "/data/data/com.exteragram.messenger/cache"
        current_time = time.time()
        
        if os.path.exists(temp_dir):
            for filename in os.listdir(temp_dir):
                if filename.startswith("mandre_"):
                    file_path = os.path.join(temp_dir, filename)
                    if os.path.getctime(file_path) < current_time - 3600:
                        os.remove(file_path)
        
        self.log("Временные файлы очищены")
        
    except Exception as e:
        self.log(f"Ошибка очистки: {e}")
```

Важные моменты:
- Задачи выполняются в фоновом потоке, не блокируя UI.
- При выгрузке плагина все его задачи автоматически отменяются.
- Поддерживается неограниченное количество задач на плагин.

---

## 11. Команды (Commands)

MandreLib может перехватывать команды (сообщения, начинающиеся с префикса, типа `.команда`).

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
    # MandreLib автоматически обработает команды
    return Mandre.handle_outgoing_command(params) or HookResult()

def cmd_hello(self, plugin, args, params):
    """Использование: .hello"""
    BulletinHelper.show_info("Привет!")
    return None  # Вернуть None — сообщение не будет изменено, но будет отменено

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

def cmd_notify(self, plugin, args, params):
    """Показать уведомление"""
    if not args:
        Mandre.Notification.show_simple("Test", "Это тестовое уведомление! 🚀")
    else:
        Mandre.Notification.show_dialog("Demo", args, "https://i.postimg.cc/436ngppG/image.png")
    return HookResult(strategy=HookStrategy.CANCEL)
```

Префикс команды настраивается в параметрах MandreLib (по умолчанию `.`).

---

## 12. Локализация (Translations)

MandreLib может автоматически переводить строки в плагине на язык пользователя.

```python
# Словарь переводов
_translations = {
    "greeting": "Привет, {name}!",
    "error": "Произошла ошибка",
    "success": "Готово!",
    "button_save": "Сохранить",
    "button_cancel": "Отмена"
}

def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Регистрируем переводы
    # "ru" — исходный язык плагина
    Mandre.register_localizations(self, "ru", _translations)

def some_function(self):
    # Используем переводы
    greeting = Mandre.t(self, "greeting", name="Алиса")
    self.log(greeting)  # Автоматически переведётся на язык пользователя
    
    error_msg = Mandre.t(self, "error")
    BulletinHelper.show_error(error_msg)
    
    # Сохранить/Отмена на кнопках
    save_text = Mandre.t(self, "button_save")
    cancel_text = Mandre.t(self, "button_cancel")
```

### Продвинутое использование локализации

```python
def create_localized_settings(self):
    return [
        Header(text=Mandre.t(self, "settings_title")),
        Text(
            text=Mandre.t(self, "settings_description"),
            icon="msg_info_solar"
        ),
        Divider(text=Mandre.t(self, "appearance_section")),
        Text(
            text=Mandre.t(self, "theme_option"),
            icon="msg_palette_solar",
            on_click=lambda _: self._toggle_theme()
        ),
        Text(
            text=Mandre.t(self, "language_option"), 
            icon="msg_language_solar",
            on_click=lambda _: self._change_language()
        ),
        Divider(text=Mandre.t(self, "data_section")),
        Text(
            text=Mandre.t(self, "export_data"),
            icon="msg_upload_solar",
            accent=True,
            on_click=lambda _: self._export_data()
        ),
        Text(
            text=Mandre.t(self, "import_data"),
            icon="msg_download_solar", 
            accent=True,
            on_click=lambda _: self._import_data()
        )
    ]
```

MandreLib автоматически определяет язык устройства, запрашивает перевод через AI (Pollinations.AI) и кэширует его.

---

## 13. Deep linking (ссылки tg://)

Регистрировать пользовательские ссылки типа `tg://мой_плагин`.

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    # Регистрируем алиас
    Mandre.add_tg_alias("my_plugin/action", self.handle_deeplink)
    
    # Или автоматически для открытия настроек
    Mandre.register_settings_alias(self)

def handle_deeplink(self, intent):
    uri_str = str(intent.getData())
    self.log(f"Открыта ссылка: {uri_str}")
    
    # Парсим параметры из URL
    if "action=backup" in uri_str:
        self._create_backup()
    elif "action=restore" in uri_str:
        self._show_restore_dialog()

def on_plugin_unload(self):
    Mandre.remove_tg_alias("my_plugin/action")
```

### Сложная обработка Deep Links

```python
def handle_complex_deeplink(self, intent):
    uri = intent.getData()
    path = uri.getPath()
    query_params = uri.getQueryParameterNames()
    
    if path == "/dashboard":
        self._open_dashboard()
    elif path == "/settings":
        Mandre.apply_and_refresh_settings(self)
    elif path.startswith("/reminder/"):
        # Извлекаем ID напоминания из URL
        reminder_id = path.split("/")[-1]
        self._show_reminder(reminder_id)
    
    # Обрабатываем query параметры
    if "auto_open" in query_params:
        auto_param = uri.getQueryParameter("auto_open")
        if auto_param == "true":
            self._enable_auto_open()

# Регистрация множественных алиасов
def setup_deeplinks(self):
    Mandre.add_tg_alias("my_plugin/dashboard", self._open_dashboard)
    Mandre.add_tg_alias("my_plugin/backup", self._create_backup)
    Mandre.add_tg_alias("my_plugin/restore", self._show_restore_dialog)
    Mandre.add_tg_alias("my_plugin/reminder/", self._show_reminder)
```

---

## Полный пример: продвинутый плагин с всеми функциями

Вот пример плагина, который использует все возможности MandreLib 1.6.6:

```python
__id__ = "advanced_demo"
__name__ = "Продвинутая демонстрация"
__version__ = "2.0"
__author__ = "@you"
__description__ = "Демонстрация всех возможностей MandreLib 1.6.6"
__min_version__ = "11.9.0"

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Text, Divider, Input
from ui.bulletin import BulletinHelper
from mandre_lib import Mandre
import time
import tempfile
import json
import os
import zipfile
from datetime import datetime
from android.os import Environment

class AdvancedDemoPlugin(BasePlugin):
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        self.add_on_send_message_hook()
        
        # Устанавливаем дополнительные пакеты
        Mandre.Pip.ensure_ready()
        
        # Регистрируем команды
        Mandre.register_command(self, "device", self.cmd_device)
        Mandre.register_command(self, "share", self.cmd_share)
        Mandre.register_command(self, "notify", self.cmd_notify)
        Mandre.register_command(self, "tts", self.cmd_tts)
        Mandre.register_command(self, "auth", self.cmd_auth)
        Mandre.register_command(self, "report", self.cmd_report)
        
        # Настраиваем автоматическую локализацию
        translations = {
            "hello": "Привет! 👋",
            "device_info": "📱 Информация об устройстве",
            "create_report": "📊 Создать отчёт",
            "send_notification": "🔔 Отправить уведомление"
        }
        Mandre.register_localizations(self, "ru", translations)
        
        # Настраиваем планировщик задач
        Mandre.schedule_task(self, "periodic_check", 60, self.periodic_check)
        
        self.log("Advanced Demo загружен")
    
    def on_send_message_hook(self, params):
        return Mandre.handle_outgoing_command(params) or HookResult()
    
    def cmd_device(self, plugin, args, params):
        """Показать информацию об устройстве"""
        simple_info = Mandre.Device.get_simple_info()
        params["message"] = f"📱 {simple_info}"
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    def cmd_share(self, plugin, args, params):
        """Поделиться файлом с данными об устройстве"""
        device_info = Mandre.Device.get_device_info()
        
        with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
            json.dump(device_info, f, ensure_ascii=False, indent=2)
            temp_file_path = f.name
        
        Mandre.Share.share_file(temp_file_path, "Информация об устройстве")
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_notify(self, plugin, args, params):
        """Показать уведомление"""
        if not args:
            Mandre.Notification.show_simple("Advanced Demo", "Это простое уведомление! 🚀")
        else:
            Mandre.Notification.show_dialog("Advanced Demo", args, "https://i.postimg.cc/436ngppG/image.png")
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_tts(self, plugin, args, params):
        """Озвучить текст"""
        if not args:
            text = "Привет! Это демонстрация текст-в-речь от MandreLib!"
        else:
            text = args
        MandreTTS.speak(text)
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_auth(self, plugin, args, params):
        """Запросить аутентификацию"""
        MandreAuth.request(
            on_success=lambda: BulletinHelper.show_success("Аутентификация успешна! ✅"),
            on_failure=lambda: BulletinHelper.show_error("Аутентификация не удалась! ❌"),
            title="Проверка безопасности",
            description="Подтвердите свою личность для доступа"
        )
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_report(self, plugin, args, params):
        """Создать HTML отчёт"""
        self.create_html_report()
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def periodic_check(self):
        """Периодическая проверка состояния"""
        try:
            device_info = MandreDevice.get_device_info()
            
            # Проверяем заряд батареи (если доступно)
            battery_level = self._get_battery_level()
            if battery_level and battery_level < 20:
                Mandre.Notification.show_simple(
                    "Низкий заряд батареи", 
                    f"Заряд: {battery_level}%"
                )
            
            self.log(f"Периодическая проверка выполнена. Заряд: {battery_level}%")
        except Exception as e:
            self.log(f"Ошибка периодической проверки: {e}")
    
    def _get_battery_level(self):
        """Получить уровень заряда батареи"""
        try:
            from android.content import IntentFilter
            from android.os import BatteryManager
            
            battery_status = ApplicationLoader.applicationContext.registerReceiver(
                None, 
                IntentFilter(IntentFilter.ACTION_BATTERY_CHANGED)
            )
            if battery_status:
                level = battery_status.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
                scale = battery_status.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
                battery_pct = (level / float(scale)) * 100
                return int(battery_pct)
        except Exception:
            pass
        return None
    
    def create_html_report(self):
        """Создать HTML отчёт и отправить как изображение"""
        device_info = MandreDevice.get_device_info()
        battery_level = self._get_battery_level()
        
        html = f"""
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <style>
                body {{ 
                    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
                    margin: 0; 
                    padding: 20px; 
                    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                    color: white;
                }}
                .container {{ 
                    background: rgba(255,255,255,0.1); 
                    backdrop-filter: blur(10px);
                    border-radius: 20px; 
                    padding: 30px; 
                    box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
                }}
                h1 {{ 
                    text-align: center; 
                    margin-bottom: 30px; 
                    font-size: 2.5em;
                    text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
                }}
                .info-grid {{ 
                    display: grid; 
                    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); 
                    gap: 20px; 
                }}
                .info-card {{ 
                    background: rgba(255,255,255,0.1); 
                    border-radius: 15px; 
                    padding: 20px; 
                    border: 1px solid rgba(255,255,255,0.2);
                }}
                .label {{ 
                    font-weight: bold; 
                    color: #FFD700; 
                    font-size: 1.1em;
                }}
                .value {{ 
                    margin-top: 5px; 
                    font-size: 1.2em; 
                }}
                .timestamp {{ 
                    text-align: center; 
                    margin-top: 30px; 
                    font-size: 1.1em; 
                    opacity: 0.8;
                }}
                .icon {{ 
                    font-size: 2em; 
                    margin-bottom: 10px; 
                }}
            </style>
        </head>
        <body>
            <div class="container">
                <h1>📱 Системный отчёт</h1>
                <div class="info-grid">
                    <div class="info-card">
                        <div class="icon">🏭</div>
                        <div class="label">Устройство</div>
                        <div class="value">{device_info.get('manufacturer', 'Unknown')} {device_info.get('model', 'Unknown')}</div>
                    </div>
                    
                    <div class="info-card">
                        <div class="icon">🤖</div>
                        <div class="label">Android</div>
                        <div class="value">{device_info.get('android_version', 'Unknown')} (API {device_info.get('api_level', 'Unknown')})</div>
                    </div>
                    
                    <div class="info-card">
                        <div class="icon">💾</div>
                        <div class="label">Память</div>
                        <div class="value">{device_info.get('total_memory_mb', 'Unknown')} МБ общая</div>
                    </div>
                    
                    <div class="info-card">
                        <div class="icon">🔋</div>
                        <div class="label">Заряд батареи</div>
                        <div class="value">{battery_level or 'Unknown'}%</div>
                    </div>
                </div>
                
                <div class="timestamp">
                    Сгенерировано: {datetime.now().strftime('%d.%m.%Y %H:%M:%S')}
                </div>
            </div>
        </body>
        </html>
        """
        
        def on_result(success: bool, result_path: str):
            if success:
                MandreSend.png(result_path, "📱 Системный отчёт")
                BulletinHelper.show_success("Отчёт готов!")
            else:
                BulletinHelper.show_error(f"Ошибка создания отчёта: {result_path}")
        
        MandreWeb.render_html_to_png(html, on_result, file_prefix="system_report_")
    
    def create_settings(self):
        return [
            Header(text="Продвинутая демонстрация MandreLib 1.6.6"),
            Text(
                text=Mandre.t(self, "hello"),
                icon="msg_info_solar"
            ),
            Divider(text="🎛️ Управление устройством"),
            Text(
                text=Mandre.t(self, "device_info"),
                icon="msg_info_solar",
                on_click=lambda _: self._show_device_info()
            ),
            Text(
                text="🔋 Проверить заряд батареи",
                icon="msg_battery",
                on_click=lambda _: self._check_battery()
            ),
            Divider(text="📊 Отчёты"),
            Text(
                text=Mandre.t(self, "create_report"),
                icon="msg_filehq_solar",
                on_click=lambda _: self.create_html_report()
            ),
            Text(
                text="📈 Создать график",
                icon="msg_chart",
                on_click=lambda _: self._create_chart()
            ),
            Divider(text="🔔 Уведомления"),
            Text(
                text=Mandre.t(self, "send_notification"),
                icon="msg_notifications_solar",
                on_click=lambda _: Mandre.Notification.show_dialog(
                    "Demo", 
                    "Привет от продвинутого демо! 🚀",
                    "https://i.postimg.cc/436ngppG/image.png"
                )
            ),
            Divider(text="🎵 Аудио"),
            Text(
                text="🗣️ Озвучить приветствие",
                icon="msg_mic",
                on_click=lambda _: MandreTTS.speak("Привет! Это MandreLib в действии!")
            ),
            Divider(text="🔒 Безопасность"),
            Text(
                text="🔐 Проверка аутентификации",
                icon="msg_lock",
                on_click=lambda _: MandreAuth.request(
                    on_success=lambda: BulletinHelper.show_success("✅ Аутентификация успешна!"),
                    on_failure=lambda: BulletinHelper.show_error("❌ Аутентификация не удалась!"),
                    title="Безопасность",
                    description="Подтвердите свою личность"
                )
            ),
            Divider(text="📁 Работа с данными"),
            Text(
                text="📤 Экспорт настроек",
                icon="msg_upload_solar",
                on_click=lambda _: self._export_settings()
            ),
            Text(
                text="📥 Импорт настроек",
                icon="msg_download_solar",
                on_click=lambda _: self._import_settings()
            ),
            Divider(text="⚙️ Система"),
            Text(
                text="🔄 Обновить планировщик",
                icon="msg_reload",
                on_click=lambda _: self._refresh_scheduler()
            ),
            Text(
                text="🗑️ Очистить логи",
                icon="msg_delete",
                red=True,
                on_click=lambda _: self._clear_logs()
            )
        ]
    
    def _show_device_info(self):
        info = MandreDevice.get_device_info()
        if "error" in info:
            BulletinHelper.show_error(f"Ошибка: {info['error']}")
            return
        
        # Показываем диалог с детальной информацией
        info_text = f"""📱 Информация об устройстве

🏭 Производитель: {info.get('manufacturer', 'Unknown')}
📱 Модель: {info.get('model', 'Unknown')}
🤖 Android: {info.get('android_version', 'Unknown')} (API {info.get('api_level', 'Unknown')})
💾 Память: {info.get('total_memory_mb', 'Unknown')} МБ
🔒 Root: {'✅' if info.get('is_rooted') else '❌'}
⚠️ Эмулятор: {'✅' if info.get('is_emulator') else '❌'}"""
        
        MandreUI.show(
            title="Информация об устройстве",
            items=[info_text],
            on_select=lambda i, t: None,
            message="Подробная информация о вашем устройстве",
            cancel_text="Закрыть"
        )
    
    def _check_battery(self):
        battery_level = self._get_battery_level()
        if battery_level is None:
            BulletinHelper.show_error("Не удалось получить информацию о батарее")
            return
        
        if battery_level < 20:
            Mandre.Notification.show_simple(
                "Низкий заряд батареи", 
                f"Заряд: {battery_level}%. Рекомендуется зарядить устройство."
            )
        else:
            Mandre.Notification.show_simple(
                "Заряд батареи", 
                f"Заряд: {battery_level}%. ✅"
            )
    
    def _create_chart(self):
        html = """
        <canvas id="myChart" width="800" height="400"></canvas>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        <script>
            const ctx = document.getElementById('myChart').getContext('2d');
            new Chart(ctx, {
                type: 'line',
                data: {
                    labels: ['Янв', 'Фев', 'Мар', 'Апр', 'Май', 'Июн'],
                    datasets: [{
                        label: 'Использование ресурсов',
                        data: [65, 59, 80, 81, 56, 72],
                        borderColor: 'rgb(75, 192, 192)',
                        backgroundColor: 'rgba(75, 192, 192, 0.2)',
                        tension: 0.1
                    }, {
                        label: 'Производительность',
                        data: [28, 48, 40, 19, 86, 67],
                        borderColor: 'rgb(255, 99, 132)',
                        backgroundColor: 'rgba(255, 99, 132, 0.2)',
                        tension: 0.1
                    }]
                },
                options: {
                    responsive: true,
                    scales: {
                        y: {
                            beginAtZero: true
                        }
                    }
                }
            });
        </script>
        """
        
        def on_ready(success, path):
            if success:
                MandreSend.png(path, "📈 График производительности")
                BulletinHelper.show_success("График отправлен!")
        
        MandreWeb.render_html_to_png(html, on_ready, file_prefix="performance_chart_")
    
    def _export_settings(self):
        settings = {
            "version": self.version,
            "export_time": datetime.now().isoformat(),
            "settings": {
                "command_prefix": self.get_setting("command_prefix", "."),
                "theme": self.get_setting("theme", "default"),
                "notifications": self.get_setting("notifications", True)
            }
        }
        
        with tempfile.NamedTemporaryFile(mode='w', suffix='.json', delete=False, encoding='utf-8') as f:
            json.dump(settings, f, ensure_ascii=False, indent=2)
            temp_path = f.name
        
        Mandre.Share.share_file(temp_path, f"Настройки {self.name}")
        BulletinHelper.show_success("Настройки экспортированы!")
    
    def _import_settings(self):
        # Используем встроенную функцию импорта из MandreLib
        try:
            self._handle_import_data(self.id)
        except Exception as e:
            BulletinHelper.show_error(f"Ошибка импорта: {e}")
    
    def _refresh_scheduler(self):
        # Перезапускаем все задачи
        Mandre.cancel_task(self, "periodic_check")
        Mandre.schedule_task(self, "periodic_check", 60, self.periodic_check)
        BulletinHelper.show_success("Планировщик обновлён!")
    
    def _clear_logs(self):
        try:
            import os
            log_files = [
                "/data/data/com.exteragram.messenger/files/plugins/mandre_lib_data/mandrelib/pip.log"
            ]
            
            for log_file in log_files:
                if os.path.exists(log_file):
                    os.remove(log_file)
            
            BulletinHelper.show_success("Логи очищены!")
        except Exception as e:
            BulletinHelper.show_error(f"Ошибка очистки логов: {e}")
```

---

## Частые ошибки и как их избежать

**1. Забыли активировать персистентное хранилище**
```python
# ❌ Неправильно
def on_plugin_load(self):
    self.set_setting("key", "value")  # Данные потеряются

# ✅ Правильно
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    self.set_setting("key", "value")  # Сохранится
```

**2. UI-операции из фонового потока**
```python
from android_utils import run_on_ui_thread
from client_utils import run_on_queue

# ❌ Неправильно
run_on_queue(lambda: BulletinHelper.show_info("Message"))

# ✅ Правильно
def background_task():
    result = "Готово!" # some_long_operation()
    run_on_ui_thread(lambda: BulletinHelper.show_info(result))

run_on_queue(background_task)
```

**3. Бесконечные задачи**
```python
# ❌ Неправильно
def on_plugin_load(self):
    # Задача будет повторяться вечно даже после выгрузки
    Mandre.schedule_task(self, "infinite", 1, self.do_something)

# ✅ Правильно
def on_plugin_unload(self):
    Mandre.cancel_task(self, "infinite")
```

**4. Неправильная работа с временными файлами**
```python
# ❌ Неправильно
def share_data(self):
    file_path = "/tmp/data.txt"  # Может не существовать или не быть доступа
    Mandre.Share.share_file(file_path, "Data")

# ✅ Правильно
def share_data(self):
    import tempfile
    with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
        f.write("My data")
        temp_path = f.name
    
    Mandre.Share.share_file(temp_path, "Data")
    # Файл автоматически удалится через 5 минут
```

**5. Забыли обработать ошибки при работе с устройством**
```python
# ❌ Неправильно
def get_info(self):
    info = Mandre.Device.get_device_info()
    self.log(info['manufacturer'])  # Может упасть, если есть ключ "error"

# ✅ Правильно
def get_info(self):
    info = Mandre.Device.get_device_info()
    if "error" in info:
        BulletinHelper.show_error(f"Ошибка: {info['error']}")
        return
    self.log(info['manufacturer'])
```

**6. Неправильное использование PIP**
```python
# ❌ Неправильно - попытка импорта без установки
import requests
response = requests.get("https://api.example.com")

# ✅ Правильно - установка и импорт
try:
    import requests
except ImportError:
    Mandre.Pip.install("requests")
    requests = Mandre.Pip.import_module("requests")

response = requests.get("https://api.example.com")
```

**7. Игнорирование потокобезопасности**
```python
# ❌ Неправильно - работа с UI из фонового потока
def background_check(self):
    device_info = MandreDevice.get_device_info()
    BulletinHelper.show_info(f"Device: {device_info['model']}")  # Может крашнуть

# ✅ Правильно - UI операции в UI потоке
def background_check(self):
    device_info = MandreDevice.get_device_info()
    run_on_ui_thread(lambda: BulletinHelper.show_info(f"Device: {device_info['model']}"))
```

---

## Полный справочник API

### Mandre (главный класс)

| Метод | Описание |
|-------|----------|
| `use_persistent_storage(plugin)` | Активирует автосохранение настроек |
| `schedule_task(plugin, name, interval, callback)` | Запускает повторяющуюся задачу |
| `cancel_task(plugin, name)` | Отменяет задачу |
| `register_command(plugin, name, callback)` | Регистрирует команду |
| `handle_outgoing_command(params)` | Обрабатывает команды в хуке |
| `add_tg_alias(path, callback)` | Регистрирует tg:// ссылку |
| `remove_tg_alias(path)` | Удаляет tg:// ссылку |
| `register_settings_alias(plugin)` | Автоматическая ссылка на настройки |
| `apply_and_refresh_settings(plugin)` | Перезагружает настройки |
| `register_localizations(plugin, lang, dict)` | Регистрирует переводы |
| `t(plugin, key, **kwargs)` | Получает переведённую строку |
| `sql_get_database()` | Получает объект БД MessagesStorage |
| `sql_init_kv(plugin_id, table_name)` | Создаёт KV-таблицу |
| `sql_kv_set(plugin_id, key, value, table_name)` | Записывает значение в KV |
| `sql_kv_get(plugin_id, key, table_name)` | Читает значение из KV |
| `sql_kv_get_int(plugin_id, key, default, table_name)` | Читает int значение |
| `sql_kv_delete_prefix(plugin_id, prefix, table_name)` | Удаляет ключи с префиксом |
| `register_synthetic_channel(channel_id, title, megagroup, broadcast)` | Регистрирует синтетический канал |

### MandreData (хранилище)

| Метод | Описание |
|-------|----------|
| `write_persistent_json(plugin_id, filename, data)` | Сохраняет JSON |
| `read_persistent_json(plugin_id, filename, default)` | Читает JSON |
| `get_persistent_path(plugin_id, filename)` | Получает путь к файлу |
| `list_persistent_plugins()` | Список плагинов с данными |
| `list_files_for_plugin(plugin_id)` | Список файлов плагина |
| `delete_persistent_plugin_data(plugin_id)` | Удаляет все данные плагина |

### MandreUI (интерфейс)

| Метод | Описание |
|-------|----------|
| `show(title, items, on_select, message, cancel_text)` | Показывает диалог выбора |
| `select_chat(title, on_select, search_hint, cancel_text)` | Селектор чатов |
| `ripple(intensity, vibrate)` | Эффект волны |
| `setup_settings_bottom_bar(plugin, items, ...)` | Настраивает нижнюю панель |
| `update_bottom_bar(plugin_id, active_index)` | Обновляет активную вкладку |

### MandreShare (отправка)

| Метод | Описание |
|-------|----------|
| `share_text(text, title)` | Отправляет текст |
| `share_file(file_path, title, mime_type)` | Отправляет файл |

### MandreSend (прямая отправка)

| Метод | Описание |
|-------|----------|
| `png(path, caption)` | Отправляет PNG изображение в текущий чат |

### MandreDevice (устройство)

| Метод | Описание |
|-------|----------|
| `get_device_info()` | Полная информация об устройстве (dict) |
| `get_simple_info()` | Краткая информация (строка) |

### MandreNotification (уведомления)

| Метод | Описание |
|-------|----------|
| `show_simple(title, text, channel_id)` | Простое уведомление |
| `show_dialog(sender_name, message, avatar_url, channel_id)` | Уведомление в стиле диалога |

### MandreTTS (речь)

| Метод | Описание |
|-------|----------|
| `speak(text)` | Озвучивает текст |

### MandreAuth (аутентификация)

| Метод | Описание |
|-------|----------|
| `request(on_success, on_failure, title, description)` | Запрашивает аутентификацию |

### MandrePip (PIP менеджер)

| Метод | Описание |
|-------|----------|
| `ensure_ready()` | Инициализирует PIP |
| `pip(args)` | Выполняет PIP команду |
| `install(spec)` | Устанавливает пакет |
| `import_module(mod)` | Импортирует модуль после установки |

### MandreWeb (веб-рендеринг)

| Метод | Описание |
|-------|----------|
| `render_html_to_png(html, on_result, width, height, ...)` | Рендерит HTML в PNG |

---

## Практические примеры

### Пример 1: Система мониторинга устройства

```python
class DeviceMonitorPlugin(BasePlugin):
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # Настраиваем мониторинг каждые 5 минут
        Mandre.schedule_task(self, "device_monitor", 300, self.monitor_device)
        
        # Регистрируем команду для получения отчёта
        Mandre.register_command(self, "status", self.cmd_status)
    
    def monitor_device(self):
        device_info = MandreDevice.get_device_info()
        
        # Проверяем температуру процессора (пример)
        cpu_temp = self._get_cpu_temperature()
        if cpu_temp and cpu_temp > 70:
            Mandre.Notification.show_simple(
                "⚠️ Перегрев",
                f"Температура CPU: {cpu_temp}°C. Рекомендуется охладить устройство.",
                channel_id="thermal_warnings"
            )
        
        # Проверяем использование памяти
        available_memory = device_info.get('available_memory_mb', 0)
        total_memory = device_info.get('total_memory_mb', 1)
        memory_usage = ((total_memory - available_memory) / total_memory) * 100
        
        if memory_usage > 85:
            Mandre.Notification.show_simple(
                "💾 Мало памяти",
                f"Использование памяти: {memory_usage:.1f}%. Рекомендуется закрыть лишние приложения.",
                channel_id="memory_warnings"
            )
    
    def cmd_status(self, plugin, args, params):
        """Показать статус системы"""
        info = MandreDevice.get_device_info()
        battery = self._get_battery_level()
        
        status = f"""📊 Статус системы

💾 Память: {info.get('available_memory_mb', '?')} МБ свободно из {info.get('total_memory_mb', '?')} МБ
🔋 Батарея: {battery or 'Неизвестно'}%
🌡️ CPU: {self._get_cpu_temperature() or 'Неизвестно'}°C
📱 Android: {info.get('android_version', 'Unknown')}"""
        
        params["message"] = status
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    def _get_cpu_temperature(self):
        try:
            # Пример чтения температуры из файловой системы
            temp_files = ["/sys/class/thermal/thermal_zone0/temp", "/sys/devices/system/cpu/cpu0/cpufreq/cpu_temp"]
            for temp_file in temp_files:
                if os.path.exists(temp_file):
                    with open(temp_file, 'r') as f:
                        temp = int(f.read().strip())
                        if temp > 1000:  # Делим на 1000 если в миллиградусах
                            return temp // 1000
                        return temp
        except:
            pass
        return None
```

### Пример 2: Система резервного копирования

```python
class BackupManagerPlugin(BasePlugin):
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # Автоматическое резервное копирование каждый день в 3:00
        Mandre.schedule_task(self, "auto_backup", 86400, self.create_backup)
        
        # Команды управления
        Mandre.register_command(self, "backup", self.cmd_backup)
        Mandre.register_command(self, "restore", self.cmd_restore)
        Mandre.register_command(self, "list_backups", self.cmd_list_backups)
    
    def cmd_backup(self, plugin, args, params):
        """Создать резервную копию"""
        self.create_backup()
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_restore(self, plugin, args, params):
        """Восстановить из резервной копии"""
        backups = self._list_backup_files()
        if not backups:
            params["message"] = "❌ Нет доступных резервных копий"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
        backup_items = [f"{backup['name']} ({backup['size']})" for backup in backups]
        
        MandreUI.show(
            title="Выберите резервную копию для восстановления",
            items=backup_items,
            on_select=lambda i, text: self._restore_backup(backups[i]['path']),
            message="⚠️ Внимание: текущие данные будут перезаписаны!"
        )
        
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def cmd_list_backups(self, plugin, args, params):
        """Показать список резервных копий"""
        backups = self._list_backup_files()
        if not backups:
            params["message"] = "📦 Нет резервных копий"
        else:
            backup_list = "\n".join([f"📦 {backup['name']} - {backup['size']}" for backup in backups])
            params["message"] = f"📦 Доступные резервные копии:\n{backup_list}"
        
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    def create_backup(self):
        try:
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_name = f"backup_{self.id}_{timestamp}.zip"
            
            downloads_dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS)
            backup_dir = File(downloads_dir, "exteraGram_Backups")
            if not backup_dir.exists(): backup_dir.mkdirs()
            
            backup_path = os.path.join(backup_dir.getAbsolutePath(), backup_name)
            
            # Создаём ZIP архив со всеми данными плагинов MandreLib
            with zipfile.ZipFile(backup_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
                plugins_with_data = MandreData.list_persistent_plugins()
                for plugin_id in plugins_with_data:
                    plugin_dir = File(MandreData._get_base_data_dir(), plugin_id)
                    if plugin_dir.exists():
                        for filename in MandreData.list_files_for_plugin(plugin_id):
                            file_path = MandreData.get_persistent_path(plugin_id, filename)
                            zipf.write(file_path, arcname=f"{plugin_id}/{filename}")
            
            file_size = os.path.getsize(backup_path)
            BulletinHelper.show_success(f"✅ Резервная копия создана: {backup_name} ({file_size} байт)")
            
            # Отправляем уведомление
            Mandre.Notification.show_simple(
                "💾 Резервная копия готова",
                f"Создана копия: {backup_name}",
                channel_id="backup_notifications"
            )
            
        except Exception as e:
            self.log(f"Ошибка создания резервной копии: {e}")
            BulletinHelper.show_error(f"❌ Ошибка резервного копирования: {e}")
    
    def _list_backup_files(self):
        backups = []
        try:
            downloads_dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS)
            backup_dir = File(downloads_dir, "exteraGram_Backups")
            if backup_dir.exists():
                for filename in backup_dir.listFiles():
                    if filename.getName().endswith('.zip'):
                        backups.append({
                            'name': filename.getName(),
                            'path': filename.getAbsolutePath(),
                            'size': self._format_file_size(filename.length())
                        })
        except Exception as e:
            self.log(f"Ошибка получения списка резервных копий: {e}")
        return sorted(backups, key=lambda x: x['name'], reverse=True)
    
    def _restore_backup(self, backup_path):
        try:
            temp_dir = ApplicationLoader.applicationContext.getCacheDir()
            temp_zip = File(temp_dir, "restore_temp.zip")
            
            # Копируем ZIP во временную директорию
            with open(backup_path, 'rb') as src, open(temp_zip.getAbsolutePath(), 'wb') as dst:
                dst.write(src.read())
            
            # Извлекаем в базовую директорию данных
            with zipfile.ZipFile(temp_zip.getAbsolutePath(), 'r') as zipf:
                zipf.extractall(MandreData._get_base_data_dir().getAbsolutePath())
            
            temp_zip.delete()
            
            BulletinHelper.show_success("✅ Восстановление завершено успешно!")
            
            Mandre.Notification.show_simple(
                "🔄 Восстановление завершено",
                "Данные успешно восстановлены из резервной копии",
                channel_id="backup_notifications"
            )
            
            # Перезагружаем все плагины с данными
            for plugin_id in MandreData.list_persistent_plugins():
                Mandre.apply_and_refresh_settings(plugin_id)
                
        except Exception as e:
            self.log(f"Ошибка восстановления: {e}")
            BulletinHelper.show_error(f"❌ Ошибка восстановления: {e}")
    
    def _format_file_size(self, size_bytes):
        if size_bytes < 1024:
            return f"{size_bytes} Б"
        elif size_bytes < 1024 * 1024:
            return f"{size_bytes / 1024:.1f} КБ"
        else:
            return f"{size_bytes / (1024 * 1024):.1f} МБ"
```

### Пример 3: Система уведомлений с группировкой

```python
class SmartNotifierPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.notification_queue = []
        self.queue_timer = None
    
    def on_plugin_load(self):
        Mandre.use_persistent_storage(self)
        
        # Регистрируем команду для тестирования
        Mandre.register_command(self, "notify", self.cmd_notify)
        
        # Настраиваем планировщик для проверки
        Mandre.schedule_task(self, "queue_check", 5, self.check_notification_queue)
    
    def cmd_notify(self, plugin, args, params):
        """Тестовое уведомление"""
        if args:
            self.queue_notification(args)
        else:
            self.queue_notification("Тестовое уведомление без текста")
        
        return HookResult(strategy=HookStrategy.CANCEL)
    
    def queue_notification(self, message):
        """Добавляет уведомление в очередь для группировки"""
        self.notification_queue.append({
            'message': message,
            'timestamp': time.time(),
            'id': int(time.time() * 1000) % 10000
        })
        
        # Запускаем таймер если не запущен
        if not self.queue_timer:
            self.queue_timer = threading.Timer(2.0, self.process_notification_queue)
            self.queue_timer.start()
    
    def check_notification_queue(self):
        """Периодическая проверка очереди"""
        if self.notification_queue and not self.queue_timer:
            self.process_notification_queue()
    
    def process_notification_queue(self):
        """Обрабатывает очередь уведомлений"""
        if not self.notification_queue:
            self.queue_timer = None
            return
        
        count = len(self.notification_queue)
        messages = [item['message'] for item in self.notification_queue]
        
        # Группируем похожие уведомления
        if count == 1:
            Mandre.Notification.show_simple(
                "Уведомление", 
                messages[0],
                channel_id="smart_notifications"
            )
        elif count <= 5:
            # Показываем несколько уведомлений в одном
            combined_message = "\n".join([f"• {msg}" for msg in messages])
            Mandre.Notification.show_simple(
                f"Уведомления ({count})",
                combined_message,
                channel_id="smart_notifications"
            )
        else:
            # Много уведомлений - показываем сводку
            Mandre.Notification.show_simple(
                f"Много уведомлений ({count})",
                f"Поступило {count} новых уведомлений. Проверьте приложение.",
                channel_id="smart_notifications"
            )
        
        self.notification_queue.clear()
        self.queue_timer = None
        
        self.log(f"Обработано {count} уведомлений")
```

---

## Советы по оптимизации

### 1. Кэширование информации об устройстве

Информация об устройстве не меняется часто, поэтому её можно кэшировать:

```python
def on_plugin_load(self):
    Mandre.use_persistent_storage(self)
    
    last_update = self.get_setting("device_info_updated", 0)
    current_time = int(time.time())
    
    if current_time - last_update > 86400:  # 24 часа
        device_info = Mandre.Device.get_device_info()
        MandreData.write_persistent_json(self.id, "device_cache.json", device_info)
        self.set_setting("device_info_updated", current_time)

def get_cached_device_info(self):
    return MandreData.read_persistent_json(self.id, "device_cache.json", {})
```

### 2. Умные уведомления

Не спамьте пользователя — группируйте уведомления, если это уместно:

```python
# Используйте систему группировки из примера выше
class SmartNotifierPlugin
```

### 3. Оптимизация PIP операций

```python
# Устанавливайте пакеты только при необходимости
def ensure_required_packages(self):
    required_packages = {
        'requests': 'HTTP библиотека',
        'pillow': 'Работа с изображениями'
    }
    
    for package, description in required_packages.items():
        try:
            __import__(package)
        except ImportError:
            self.log(f"Установка {package} для {description}")
            Mandre.Pip.install(package)
```

### 4. Эффективное использование памяти

```python
def process_large_data(self, data_source):
    # Не загружайте большие данные в память сразу
    def process_in_chunks():
        chunk_size = 1024  # Обрабатываем по 1KB
        processed_count = 0
        
        for chunk in self._read_chunks(data_source, chunk_size):
            processed_data = self._process_chunk(chunk)
            
            # Обновляем UI только каждые 100 обработанных блоков
            if processed_count % 100 == 0:
                run_on_ui_thread(lambda: self._update_progress(processed_count))
            
            processed_count += 1
    
    run_on_queue(process_in_chunks)
```

---

## Новые возможности в версии 1.6.6

### ✨ Полноценный PIP менеджер
- Автоматическая установка Python пакетов
- Поддержка wheel файлов
- Кастомные индексы и прокси
- Кэширование и оптимизация

### 🌐 Веб-рендеринг
- HTML в PNG конвертер
- Поддержка JavaScript и CSS
- Автоматическое определение готовности страницы
- Настраиваемые размеры и параметры

### отправка картинок в чат
---

## Заключение

MandreLib 1.6.6 предоставляет всё необходимое для создания мощных и функциональных плагинов:

✅ **Персистентное хранилище** — данные сохраняются автоматически  
✅ **PIP менеджер** — установка Python пакетов  
✅ **Веб-рендеринг** — HTML в PNG конвертер  
✅ **UI компоненты** — готовые диалоги и селекторы  
✅ **Отправка файлов** — делитесь текстом и файлами любых типов  
✅ **Информация об устройстве** — полная диагностика системы  
✅ **Системные уведомления** — профессиональные уведомления как в мессенджерах  
✅ **TTS** — озвучивание текста  
✅ **Аутентификация** — защита важных операций  
✅ **Планировщик** — автоматизация задач  
✅ **Команды** — удобный CLI в Telegram  
✅ **Локализация** — автоматический перевод  
✅ **Deep linking** — пользовательские ссылки  
✅ **SQL KV хелперы** — работа с базой данных  
✅ **Синтетические каналы** — создание чатов без сети  

Используй эту документацию как справочник при разработке плагинов. Передавай её AI-ассистентам для генерации кода. Комбинируй функции для создания уникальных решений.

**Удачи в разработке! 🚀**