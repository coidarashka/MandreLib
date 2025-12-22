Introduction

Plugin development in all Telegram developers familiar language.
exteraGram Plugins

Our plugins system is powered by Chaquopy v15 and Aliucord hook.

Developers may write plugins in Python and use Xposed method hooking to change app behaviour.
Chaquopy

Chaquopy is a Java library that provides interop between Java and Python, allowing you to write plugins in Python 3.8.
Aliucord hook
Aliucord itself is a modification for the Discord Android app. We use their hook to provide Xposed functionality for plugins.


# Setup

How to start developing plugins.

## Download exteraGram

Make sure you're using the latest version of exteraGram or derivative client.

You may download latest version from the [beta channel](https://t.me/exteraGramCI).

## Enable plugins engine

After logging into your account, go to the exteraGram Preferences > Plugins and enable plugins engine.

Long-tap on header to enable developer mode.


## Next Steps

- Read the [First Plugin](first-plugin.md) guide to create your first plugin
- Explore the [Plugin Class Reference](plugin-class.md) for detailed API documentation
- Check out community plugins for inspiration and examples


# First Plugin

Running your first plugin

## Before we start

It's recommended to review the [Plugin Class Reference](plugin-class.md) documentation or keep it open for reference while developing plugins.

## Basic plugin structure

All `.plugin` files must include:

- Meta variables defined as plain strings (`__id__`, `__name__`, `__description__`, `__author__`, `__version__`, `__icon__`, `__min_version__`)
- A single class that inherits from `BasePlugin`

Here's the most basic plugin template:

```python
__id__ = "example"
__name__ = "Example Plugin"
__description__ = "A simple example plugin"
__author__ = "@username"
__version__ = "1.0"
__min_version__ = "10.14.4"

class ExamplePlugin(BasePlugin):
    def on_plugin_load(self):
        self.log("Plugin loaded!")
```

**Note:** You don't need to import `BasePlugin` as it's implicitly imported.

## Creating simple Weather plugin

In this example, we'll create a plugin that provides weather information when a user sends a message prefixed with `.wt`.

We'll use the [wttr.in](https://wttr.in) API to fetch weather data.

### Implementing network call and formatting

First, let's implement the functions to fetch and format weather data. They're quite boilerplate, so we won't look deep into it:

```python
import requests
import re

def get_weather(location):
    try:
        url = f"https://wttr.in/{location}?format=j1"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        return None

def format_weather(data, location):
    if not data or 'current_condition' not in data:
        return f"❌ Could not fetch weather for {location}"
    
    current = data['current_condition'][0]
    weather_desc = current['weatherDesc'][0]['value']
    temp_c = current['temp_C']
    temp_f = current['temp_F']
    humidity = current['humidity']
    wind_kmph = current['windspeedKmph']
    
    return f"""🌤 **Weather for {location}**
    
🌡 Temperature: {temp_c}°C ({temp_f}°F)
☁️ Condition: {weather_desc}
💧 Humidity: {humidity}%
💨 Wind: {wind_kmph} km/h"""
```

### Hooking message send event

To intercept and modify messages, we implement the `on_send_message_hook` method in our plugin class:

```python
def on_send_message_hook(self, params):
    message_text = params.get("message", "")
    
    if message_text.startswith(".wt "):
        location = message_text[4:].strip()
        if location:
            weather_data = get_weather(location)
            weather_message = format_weather(weather_data, location)
            
            params["message"] = weather_message
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    return HookResult()
```

To make your `on_send_message_hook` method actually get called by the plugin system, you need to register this hook. This is typically done in `on_plugin_load` by calling `self.add_on_send_message_hook()`.

The `on_send_message_hook` method returns a `HookResult` with a `MODIFY` strategy, which means the message will be modified before sending. An empty `HookResult` won't modify the message.

### Complete example (Initial)

Here's the complete implementation of the Weather plugin before performance enhancements:

```python
import requests
import re

__id__ = "weather"
__name__ = "Weather Plugin"
__description__ = "Get weather information using .wt command"
__author__ = "@yourname"
__version__ = "1.0"
__min_version__ = "10.14.4"

def get_weather(location):
    try:
        url = f"https://wttr.in/{location}?format=j1"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        return None

def format_weather(data, location):
    if not data or 'current_condition' not in data:
        return f"❌ Could not fetch weather for {location}"
    
    current = data['current_condition'][0]
    weather_desc = current['weatherDesc'][0]['value']
    temp_c = current['temp_C']
    temp_f = current['temp_F']
    humidity = current['humidity']
    wind_kmph = current['windspeedKmph']
    
    return f"""🌤 **Weather for {location}**
    
🌡 Temperature: {temp_c}°C ({temp_f}°F)
☁️ Condition: {weather_desc}
💧 Humidity: {humidity}%
💨 Wind: {wind_kmph} km/h"""

class WeatherPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()
    
    def on_send_message_hook(self, params):
        message_text = params.get("message", "")
        
        if message_text.startswith(".wt "):
            location = message_text[4:].strip()
            if location:
                weather_data = get_weather(location)
                weather_message = format_weather(weather_data, location)
                
                params["message"] = weather_message
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
        
        return HookResult()
```

### Testing the Plugin

Try sending message like `.wt London` in any chat. You should get something similar to this:

```
🌤 **Weather for London**

🌡 Temperature: 15°C (59°F)
☁️ Condition: Partly cloudy
💧 Humidity: 72%
💨 Wind: 8 km/h
```

## Performance Considerations

### Fixing UI freeze

You may notice that the app freezes for a few seconds when using the plugin. This happens because the network call (`requests.get`) is a blocking I/O operation running on the UI thread. While the request is processing, the app cannot render anything.

To fix this issue, move blocking calls to a separate thread or queue to avoid blocking the UI thread. We can use `client_utils.run_on_queue` for the background network request and `android_utils.run_on_ui_thread` to post results back to the UI thread (e.g., to send the message or dismiss a dialog).

Additionally, we'll show a loading indicator using `AlertDialogBuilder` from `alert.py` while fetching data and then use `client_utils.send_message` to send the processed message.

Here's the improved version:

```python
import requests
from alert import AlertDialogBuilder
from client_utils import run_on_queue, send_message, get_last_fragment
from android_utils import run_on_ui_thread

__id__ = "weather"
__name__ = "Weather Plugin"
__description__ = "Get weather information using .wt command"
__author__ = "@yourname"
__version__ = "1.1"
__min_version__ = "10.14.4"

def get_weather(location):
    try:
        url = f"https://wttr.in/{location}?format=j1"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        return None

def format_weather(data, location):
    if not data or 'current_condition' not in data:
        return f"❌ Could not fetch weather for {location}"
    
    current = data['current_condition'][0]
    weather_desc = current['weatherDesc'][0]['value']
    temp_c = current['temp_C']
    temp_f = current['temp_F']
    humidity = current['humidity']
    wind_kmph = current['windspeedKmph']
    
    return f"""🌤 **Weather for {location}**
    
🌡 Temperature: {temp_c}°C ({temp_f}°F)
☁️ Condition: {weather_desc}
💧 Humidity: {humidity}%
💨 Wind: {wind_kmph} km/h"""

class WeatherPlugin(BasePlugin):
    def __init__(self):
        self.progress_dialog_builder = None
    
    def on_plugin_load(self):
        self.add_on_send_message_hook()
    
    def on_send_message_hook(self, params):
        message_text = params.get("message", "")
        
        if message_text.startswith(".wt "):
            location = message_text[4:].strip()
            if location:
                # Show loading dialog
                context = get_last_fragment().getParentActivity()
                self.progress_dialog_builder = AlertDialogBuilder(
                    context, 
                    AlertDialogBuilder.ALERT_TYPE_SPINNER
                )
                self.progress_dialog_builder.set_message("Fetching weather data...")
                self.progress_dialog_builder.show()
                
                # Process in background
                run_on_queue(lambda: self._process_weather_request(location, params))
                
                # Cancel original message
                return HookResult(strategy=HookStrategy.CANCEL)
        
        return HookResult()
    
    def _process_weather_request(self, location, original_params):
        weather_data = get_weather(location)
        weather_message = format_weather(weather_data, location)
        
        # Send message on UI thread
        run_on_ui_thread(lambda: self._send_message_and_dismiss_dialog(
            weather_message, original_params
        ))
    
    def _send_message_and_dismiss_dialog(self, message, original_params):
        # Dismiss progress dialog
        if self.progress_dialog_builder:
            self.progress_dialog_builder.dismiss()
            self.progress_dialog_builder = None
        
        # Send the weather message
        params = {
            "peer": original_params.get("peer"),
            "message": message
        }
        send_message(params)
```

In this improved version:

- We import `AlertDialogBuilder` from `alert`.
- The `__init__` method initializes `self.progress_dialog_builder`. The `on_plugin_load` method is used to call `self.add_on_send_message_hook()`.
- When `.wt` is detected, we create and `show()` an `AlertDialogBuilder` of `ALERT_TYPE_SPINNER`.
- The actual work (`_process_weather_request`) is dispatched to a background queue using `run_on_queue`.
- `_process_weather_request` performs the network call. After getting the result, it schedules `_send_message_and_dismiss_dialog` on the UI thread using `run_on_ui_thread`.
- `_send_message_and_dismiss_dialog` dismisses the progress dialog and then uses `client_utils.send_message` to send the weather information as a new message.
- The original message sending is cancelled by returning `HookResult(strategy=HookStrategy.CANCEL)`.

This approach ensures the UI remains responsive while fetching data.

## What's next?

To further develop your plugin skills, explore community-made plugins and check out the other documentation sections:

- [Plugin Class Reference](plugin-class.md) - Detailed API documentation
- [Xposed Hooking](xposed-hooking.md) - Advanced method interception
- [Android Utils](android-utils.md) - UI utilities and Android helpers
- [Client Utils](client-utils.md) - Telegram client utilities
- [Alert Dialog Builder](alert-dialog-builder.md) - Creating dialogs
- [Bulletin Helper](bulletin-helper.md) - Showing notifications


# Plugin Class

Understand the Plugin class structure.

## Metadata

Metadata should be defined as plain strings. No concatenation or formatting, since it's parsed using AST.

Required fields: `__id__`, `__name__`, `__description__`, `__author__`, `__min_version__`.

- `__author__`: Supports plain text names or Telegram usernames/channel links (e.g., `@yourUsername` or `@yourPluginChannel`). These may be displayed as clickable links in the UI.
- `__description__`: Supports basic markdown for formatting.
- `__version__`: If not defined, your plugin will have version `1.0` by default.
- `__icon__`: To fill this field, you can use a link to your sticker pack (e.g., `https://t.me/addstickers/MyPackName`), then specify the index of the sticker, separated by `/`. The index starts from `0` (e.g., for the second sticker, use `MyPackName/1`).

Example metadata:

```python
__id__ = "my_plugin"
__name__ = "My Awesome Plugin"
__description__ = "This plugin does **amazing** things!"
__author__ = "@myusername"
__version__ = "1.2.3"
__icon__ = "MyStickers/0"
__min_version__ = "10.14.4"
```

## Settings

You may create an activity for plugin settings using `create_settings` method. This method should return a list of setting control objects.

```python
def create_settings(self):
    return [
        {
            'type': PluginsConstants.Settings.TYPE_SWITCH,
            'key': 'enable_feature',
            'title': 'Enable Feature',
            'summary': 'Toggle this awesome feature',
            'default': True
        },
        {
            'type': PluginsConstants.Settings.TYPE_TEXT,
            'key': 'api_key',
            'title': 'API Key',
            'summary': 'Enter your API key',
            'default': ''
        }
    ]
```

To access settings from the code, use `self.get_setting("KEY", DEFAULT_VALUE)` method:

```python
def some_method(self):
    is_enabled = self.get_setting("enable_feature", True)
    api_key = self.get_setting("api_key", "")
```

To save or update a setting's value programmatically, use `self.set_setting("KEY", NEW_VALUE)` method:

```python
def update_setting(self):
    self.set_setting("enable_feature", False)
    self.set_setting("api_key", "new_api_key_123")
```

The `set_setting` method will persist the new value, and it will be reflected in the settings UI if the control is bound to that key.

### Supported controls:

| Type | Description | Additional Fields |
|------|-------------|-------------------|
| `TYPE_SWITCH` | Toggle switch | `default: bool` |
| `TYPE_CHECKBOX` | Checkbox | `default: bool` |
| `TYPE_TEXT` | Text input field | `default: str`, `hint: str` |
| `TYPE_NUMERIC` | Numeric input | `default: int/float`, `min: int`, `max: int` |
| `TYPE_LIST` | List selection | `entries: List[str]`, `entry_values: List[str]`, `default: str` |
| `TYPE_MULTISELECT` | Multiple selection | `entries: List[str]`, `entry_values: List[str]`, `default: List[str]` |
| `TYPE_CATEGORY` | Settings category header | `title: str` |
| `TYPE_INFO` | Information text | `title: str`, `summary: str` |

(Note: The `type` field is usually set automatically by the system and corresponds to `PluginsConstants.Settings.TYPE_...`. Icon names are examples and depend on available resources.)

## Plugin events

### Load and unload

- `on_plugin_load` occurs when user enables the plugin or on application startup.
- `on_plugin_unload` occurs when user disables the plugin or on application shutdown.

```python
class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.log("Plugin loaded!")
        self.add_on_send_message_hook()
    
    def on_plugin_unload(self):
        self.log("Plugin unloaded!")
        # Cleanup resources here
```

### Application events

The `AppEvent` enum provides the following events:

- `START` - Application is starting
- `STOP` - Application is stopping
- `PAUSE` - Application is paused (e.g., backgrounded)
- `RESUME` - Application is resumed (e.g., brought to foreground)

```python
def on_app_event(self, event):
    if event == AppEvent.START:
        self.log("App started")
    elif event == AppEvent.PAUSE:
        self.log("App paused")
```

## Menu Items

You can add custom actions to various menus within the application, such as the context menu for messages or the action menu in a user's profile. This is done by adding a `MenuItemData` object.

### MenuItemData

To add a menu item, you call `self.add_menu_item()` with a `MenuItemData` object, which has the following properties:

- **menu_type: MenuItemType**: Required. Specifies which menu to add the item to. The available types are:
  - `MenuItemType.MESSAGE_CONTEXT_MENU`: Long-press menu on a message.
  - `MenuItemType.DRAWER_MENU`: The main navigation drawer (hamburger menu).
  - `MenuItemType.CHAT_ACTION_MENU`: The three-dot menu inside a chat screen.
  - `MenuItemType.PROFILE_ACTION_MENU`: The three-dot menu on a user, bot, or channel profile screen.

- **text: str**: Required. The text displayed for the menu item.
- **on_click: Callable[[Dict[str, Any]], None]**: Required. A function that will be called when the user taps the item. It receives a dictionary containing context-specific data.
- **item_id: str**: Optional. A unique ID for this item. Useful if you need to remove it later with `remove_menu_item()`. If not provided, a unique ID is generated.
- **icon: str**: Optional. The name of a drawable resource to use as an icon for the item (e.g., `"msg_info"`, `"msg_delete"`).
- **subtext: str**: Optional. Additional text displayed below the main text.
- **condition: str**: Optional. A MVEL expression to conditionally show the item. (e.g., `"message.isOut()"`).
- **priority: int**: Optional. A number to influence the item's position in the menu. Higher numbers appear first.

Example:

```python
def on_plugin_load(self):
    menu_item = MenuItemData(
        menu_type=MenuItemType.MESSAGE_CONTEXT_MENU,
        text="Process Message",
        on_click=self.handle_message_click,
        icon="msg_info",
        condition="!message.isOut()"
    )
    self.add_menu_item(menu_item)

def handle_message_click(self, context):
    message = context.get("message")
    if message:
        self.log(f"Processing message: {message.messageText}")
```

### The `on_click` Context

The `on_click` callback receives a dictionary with data relevant to the context where the menu was opened. The available keys depend on the `MenuItemType` and the specific situation. For example, a message context menu will provide a `message` object, while a profile menu will provide a `user` object.

It's best practice to check for the existence of a key before using it. You can log the dictionary's keys to discover what's available: `self.log(f"Context keys: {list(context.keys())}")`.

Here are some of the possible keys you might find in the context dictionary:

- **account: int**: The current user account instance number.
- **context: android.content.Context**: The Android application context.
- **fragment: org.telegram.ui.ActionBar.BaseFragment**: The current UI fragment.
- **dialog_id: long**: The dialog ID for the current chat.
- **user: TLRPC.User**: The `User` object (e.g., in a profile menu).
- **userId: long**: The ID of the `user`.
- **userFull: TLRPC.UserFull**: The `UserFull` object with more details.
- **chat: TLRPC.Chat**: The `Chat` object for a basic group or channel.
- **chatId: long**: The ID of the `chat`.
- **chatFull: TLRPC.ChatFull**: The `ChatFull` object with more details.
- **encryptedChat: TLRPC.EncryptedChat**: The object for a secret chat.
- **message: org.telegram.messenger.MessageObject**: The `MessageObject` that was clicked on.
- **groupedMessages: org.telegram.messenger.MessageObject.GroupedMessages**: Information about grouped media (albums).
- **botInfo: TL_bots.BotInfo**: Information about a bot.

### Removing Menu Items

If you provided a custom `item_id` when adding a menu item, you can remove it programmatically using `self.remove_menu_item(item_id)`. However, in most cases, this is not necessary, as all of a plugin's menu items are automatically removed when the plugin is unloaded.

## Hooks

To intercept network requests, responses, or client-side events, you first need to register a hook.

You can register hooks for specific Telegram API requests using their TL-schema name:

```python
self.add_hook("TL_messages_readHistory", match_substring: bool = False, priority: int = 0)
```

- **name**: The name of the event or request (e.g., "TL_messages_readHistory").
- **match_substring**: If `True`, the hook will trigger if `name` is a substring of the actual event/request name. Defaults to `False`.
- **priority**: Hooks with higher priority are executed first. Defaults to `0`.

Examples:

```python
self.add_hook("TL_messages_readHistory")
self.add_hook("requestCall")
self.add_hook("TL_channels_readHistory")
```

The list of names for requests could be found [here](common-source-classes.md#tlrpc-all-telegram-request-models).

For the common case of hooking message sending, you can use a helper:

```python
self.add_on_send_message_hook(priority: int = 0)
```

### API Request Hooks

These hooks allow you to inspect or modify outgoing requests and incoming responses.

```python
def pre_request_hook(self, request):
    # Called before sending a request
    self.log(f"Sending request: {type(request).__name__}")
    return HookResult()

def post_request_hook(self, request, response):
    # Called after receiving a response
    self.log(f"Received response for: {type(request).__name__}")
    return HookResult()
```

Hook results determine the action to take:

- `HookStrategy.DEFAULT`: No changes to the flow; proceed as normal.
- `HookStrategy.CANCEL`: Cancel the request (for `pre_request_hook` and `on_send_message_hook`) or suppress further processing of the response/update.
- `HookStrategy.MODIFY`: Modify the `request` (in `pre_request_hook`), `response` (in `post_request_hook`), `update` (in `on_update_hook`), `updates` (in `on_updates_hook`), or `params` (in `on_send_message_hook`). The modified object must be assigned to the corresponding field in the `HookResult` (e.g., `result.request = modified_request`).
- `HookStrategy.MODIFY_FINAL`: Same as `MODIFY`, but no other plugins hooks for this event will be called after this one.

### Update Hooks

These hooks are called when the application processes updates received from Telegram.

```python
def on_update_hook(self, update):
    # Called for each individual update
    if hasattr(update, 'message'):
        self.log(f"New message: {update.message.message}")
    return HookResult()

def on_updates_hook(self, updates):
    # Called for batch updates
    self.log(f"Received {len(updates.updates)} updates")
    return HookResult()
```

### Message Sending Hook

This hook is specifically for intercepting messages being sent by the user.

```python
def on_send_message_hook(self, params):
    message_text = params.get("message", "")
    
    if message_text.startswith("/transform"):
        # Modify the message
        params["message"] = message_text.upper()
        return HookResult(strategy=HookStrategy.MODIFY, params=params)
    
    return HookResult()
```

The `params` dictionary contains message parameters like `message`, `peer`, `replyToMsg`, etc.


# Xposed Method Hooking

Xposed method hooking to intercept and modify app behavior in your plugins.

## Introduction

Xposed method hooking allows your plugin to intercept calls to methods (or constructors) within the application, modify their parameters, change their behavior, or replace their implementation entirely. This is a powerful technique for altering app functionality at a low level.

## Hooking Concepts

To hook a method, you need to provide a "hook handler" — a Python class that defines what code to run when the target method is called. The system supports three main ways to interact with a method call.

## The Hook Handler Base Classes

For clarity and correctness, you should create your handler by inheriting from one of the abstract base classes provided in `base_plugin.py`:

- **MethodHook**: Use this when you want to run code before and/or after the original method executes, but still allow the original method to run.
- **MethodReplacement**: Use this when you want to completely replace the original method's logic with your own.

## The `param` Object

All hook callback methods receive a `param` object (`de.robv.android.xposed.XC_MethodHook.MethodHookParam`) which is your key to interacting with the method call:

- `param.thisObject`: The instance on which the method was called (`None` for static methods).
- `param.args`: A list-like object of the arguments passed to the method. You can read and modify these. Changes made in `before_hooked_method` will affect the original call.
- `param.result`: The value returned by the original method. Available in `after_hooked_method`. You can read and modify this.
- `param.method`: A `java.lang.reflect.Member` object representing the hooked method or constructor.

A special and very useful feature is `param.returnEarly = True`. If you set this in `before_hooked_method`, the original method and any `after_hooked_method` logic will be skipped entirely. You must also set `param.result` to provide an immediate return value.

**Reference**: [LSPosed XC_MethodHook.java](https://github.com/LSPosed/LSPosed/blob/master/core/src/main/java/de/robv/android/xposed/XC_MethodHook.java)

## The Hooking Process (Step-by-Step)

### 1. Find the Target Method or Constructor

First, you need a reference to the `java.lang.reflect.Method` or `java.lang.reflect.Constructor` you want to hook. This is done using Java reflection.

### 2. Implement the Hook Handler

Create a Python class that inherits from `MethodHook` or `MethodReplacement` and implements the required callback(s).

### 3. Apply the Hook

From your `BasePlugin` class, instantiate your handler and call `self.hook_method()`.

## Practical Examples

### Example 1: Modifying Arguments (Before Hook)

Let's modify every "Toast" message to add a prefix.

```python
class ToastPrefixHook(MethodHook):
    def before_hooked_method(self, param):
        if param.args and len(param.args) > 1:
            original_text = param.args[1]
            param.args[1] = f"[Plugin] {original_text}"

class MyPlugin(BasePlugin):
    def on_load(self):
        toast_class = self.java.lang.Class.forName("android.widget.Toast")
        make_text_method = toast_class.getDeclaredMethod("makeText", 
                                                       self.android.content.Context, 
                                                       self.java.lang.CharSequence, 
                                                       int)
        self.hook_method(make_text_method, ToastPrefixHook())
```

### Example 2: Changing the Return Value (After Hook)

This example hooks `BuildVars.isMainApp()` and makes it always return `False`.

```python
class BuildVarsHook(MethodHook):
    def after_hooked_method(self, param):
        param.result = False

class MyPlugin(BasePlugin):
    def on_load(self):
        build_vars_class = self.java.lang.Class.forName("org.telegram.messenger.BuildVars")
        is_main_app_method = build_vars_class.getDeclaredMethod("isMainApp")
        self.hook_method(is_main_app_method, BuildVarsHook())
```

### Example 3: Replacing a Method (MethodReplacement)

This example completely disables a specific internal logging method to reduce logcat spam.

```python
class DisableLoggingReplacement(MethodReplacement):
    def replace_hooked_method(self, param):
        # Do nothing - effectively disabling the logging
        return None

class MyPlugin(BasePlugin):
    def on_load(self):
        logger_class = self.java.lang.Class.forName("org.telegram.messenger.FileLog")
        log_method = logger_class.getDeclaredMethod("d", self.java.lang.String)
        self.hook_method(log_method, DisableLoggingReplacement())
```

## Return Values in MethodReplacement

When using `MethodReplacement`, your Python `replace_hooked_method` is the new implementation. You are responsible for returning a value of the correct type.

- For `void` Java methods, `return` or `return None`.
- For methods returning primitives (e.g., `int`, `boolean`), return a standard Python `int` or `bool`.
- For methods returning objects (e.g., `String`), return a compatible Python object or `None` (which becomes `null` in Java).


# Android Utilities

This module provides utility functions and classes for handling Android UI interactions, running code on the UI thread, and logging.

This module offers several helper classes and functions to simplify common Android development tasks within your Python plugins, such as UI updates, event handling, and logging.

## Wrappers for Java Interfaces

These classes act as convenient Python proxies for common Java functional interfaces, especially useful for setting listeners.

### R (Runnable Proxy)

A `static_proxy` class implementing Java's `java.lang.Runnable` interface. It's primarily used with `run_on_ui_thread` and can also be passed to many internal Telegram methods or other Android APIs that expect a `Runnable`.

```python
from android_utils import R

# Example usage
def my_ui_task():
    # UI update code here
    pass

runnable = R(my_ui_task)
```

Using `R` is generally preferred over creating a `dynamic_proxy` for `Runnable` due to its optimized nature as a `static_proxy`.

### OnClickListener

A `dynamic_proxy` wrapper for Android's `android.view.View.OnClickListener`. Simplifies setting click listeners on UI views from Python.

```python
from android_utils import OnClickListener

def handle_click():
    # Handle click event
    pass

listener = OnClickListener(handle_click)
some_view.setOnClickListener(listener)
```

The lambda or function passed to `OnClickListener` will be executed when the view is clicked. It takes no arguments.

### OnLongClickListener

A `dynamic_proxy` wrapper for Android's `android.view.View.OnLongClickListener`. Used for handling long-press events on UI views.

```python
from android_utils import OnLongClickListener

def handle_long_click():
    # Handle long click event
    return True  # Return True to consume the event

listener = OnLongClickListener(handle_long_click)
some_view.setOnLongClickListener(listener)
```

The function passed to `OnLongClickListener` should return `True` if the long click event was consumed (preventing further processing, like a normal click), or `False` otherwise. It takes no arguments.

## Utility Functions

### run_on_ui_thread

Schedules and runs the provided Python callable on the main Android UI thread. This is crucial for any operations that modify the user interface, as UI updates must happen on this thread.

```python
from android_utils import run_on_ui_thread

def update_ui():
    # UI update code here
    pass

# Execute immediately on UI thread
run_on_ui_thread(update_ui)

# Execute with 500ms delay on UI thread
run_on_ui_thread(update_ui, delay=500)
```

**Parameters:**
- `func`: The Python callable to execute.
- `delay` (optional): Delay in milliseconds before the callable is executed. Defaults to `0` (execute as soon as possible).

### log

A versatile logging function that sends output to Android's logcat, viewable with `adb logcat` or Android Studio's Logcat panel. It intelligently handles different data types.

```python
from android_utils import log

# Simple data types
log("Hello, world!")
log(42)
log(3.14)
log(True)
log(None)

# Complex objects (will be logged with detailed structure)
log(some_complex_object)
log({"key": "value", "nested": {"data": [1, 2, 3]}})
```

**Behavior:**
- If `data` is a simple type (`str`, `int`, `float`, `bool`, or `None`), it's converted to a string and logged.
- If `data` is any other object (e.g., a complex class instance, a list, a dictionary), its detailed structure or relevant information in JSON format (via `AppUtils.printObjectDetails`) is logged. This is very useful for inspecting the state of Java or Python objects.


# Client Utilities

This module provides utility functions and classes for asynchronous tasks, making API requests, sending messages, and displaying UI notifications like bulletins.

This module contains helpers for interacting with Telegram's core functionalities, managing background tasks, and providing user feedback.

## Queues (Background Threads)

For performing long-running or blocking operations (like network requests or heavy computations) without freezing the UI, you should run your functions on a background thread. `client_utils` provides `run_on_queue` for this.

```python
from client_utils import run_on_queue, QUEUE_IO

def background_task():
    # Heavy computation or network request
    pass

# Run on IO queue immediately
run_on_queue(background_task, QUEUE_IO)

# Run with 1000ms delay
run_on_queue(background_task, QUEUE_IO, delay=1000)
```

You can specify which queue to use and add a delay (in milliseconds):

**Available Queues (as string constants):** These allow you to target specific Telegram dispatch queues.

```python
from client_utils import QUEUE_IO, QUEUE_MAIN, QUEUE_SEARCH, QUEUE_BACKGROUND

# Available queue constants:
# QUEUE_IO - For I/O operations
# QUEUE_MAIN - Main processing queue
# QUEUE_SEARCH - For search operations
# QUEUE_BACKGROUND - For background tasks
```

To get a direct Java `org.telegram.messenger.DispatchQueue` instance:

```python
from client_utils import get_queue

io_queue = get_queue(QUEUE_IO)
```

## Utilities

### Sending Telegram API Requests

To send raw Telegram API requests (TLObjects), use `send_request`. This function handles sending the request via the current account's connection manager and invoking your callback upon response or error.

```python
from client_utils import send_request, RequestCallback

def handle_response(response, error):
    if error:
        log(f"Request failed: {error}")
    else:
        log(f"Response received: {response}")

# Create a TL request object
request = SomeTLRPCRequest()

# Send the request
send_request(request, RequestCallback(handle_response))
```

`RequestCallback` is a `dynamic_proxy` for `org.telegram.tgnet.RequestDelegate`, simplifying callback implementation in Python.

### Sending Messages

The `send_message` utility simplifies sending various types of messages. It takes a dictionary of parameters.

```python
from client_utils import send_message

# Send a simple text message
send_message({
    "peer": chat_id,
    "message": "Hello, world!",
})

# Send a message with reply
send_message({
    "peer": chat_id,
    "message": "This is a reply",
    "replyToMsg": reply_to_message_id,
})

# Send a scheduled message
send_message({
    "peer": chat_id,
    "message": "Scheduled message",
    "scheduleDate": timestamp,
})
```

This function internally handles execution on the UI thread, so you don't need to wrap calls to `send_message` in `run_on_ui_thread`.

A comprehensive list of supported parameters for the `params` dictionary can be inferred from the fields of `org.telegram.messenger.SendMessagesHelper.SendMessageParams`. You can find its definition [here (SendMessagesHelper.java#L10185)](https://github.com/DrKLO/Telegram/blob/17067dfc6a1f69618a006b14e1741b75c64b276a/TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java#L10185). Common parameters include `peer`, `message`, `replyToMsg`, `entities`, `scheduleDate`, etc.

#### Sending Files

For sending files (photos, videos, documents not already uploaded), you'll typically need to use the Java methods of `SendMessagesHelper` directly, often involving preparing the file path, creating `TLRPC.InputMedia` objects, etc. The `send_message` Python utility is more for messages where media is already represented by an ID or for simpler text-based messages.

### Displaying Bulletins (Bottom Notifications)

Bulletins are small, non-intrusive notifications shown at the bottom of the screen. The `BulletinHelper` class provides an easy way to show them.

```python
from client_utils import BulletinHelper

# Show info bulletin
BulletinHelper.show_info("Information message")

# Show error bulletin
BulletinHelper.show_error("Error occurred")

# Show success bulletin
BulletinHelper.show_success("Operation completed")
```

For detailed information and examples on how to use various types of bulletins, please refer to the [Bulletin Helper documentation](/docs/bulletin-helper).

## Accessing Controllers and Managers

`client_utils.py` provides convenient getter functions for accessing various core Telegram controllers, managers, and configurations for the currently selected account.

```python
from client_utils import (
    get_messages_controller,
    get_messages_storage,
    get_send_messages_helper,
    get_connection_manager,
    get_account_instance,
    get_user_config,
    get_last_fragment
)

# Get the messages controller
messages_controller = get_messages_controller()

# Get the messages storage
messages_storage = get_messages_storage()

# Get the send messages helper
send_helper = get_send_messages_helper()

# Get the connection manager
connection_manager = get_connection_manager()

# Get the current account instance
account = get_account_instance()

# Get user configuration
user_config = get_user_config()

# Get the last active fragment
fragment = get_last_fragment()
```

These functions simplify access to key components of the Telegram client.


# Markdown Parser

This module provides the ability to parse markdown-formatted text and convert formatting entities to TLRPC objects suitable for the Telegram API.

The `markdown_utils.py` module allows you to easily convert text with common Markdown V2-style formatting into a plain text string and a list of `TLRPC.MessageEntity` objects. These entities can then be used with `client_utils.send_message` or other API methods that accept formatted text.

## Core Components

The parser returns a `ParsedMessage` object, which has two main attributes:

- `text: str`: The plain text content with all Markdown markers removed.
- `entities: Tuple[RawEntity, ...]`: A tuple of `RawEntity` objects, each representing a formatting instruction.

Each `RawEntity` object contains:

- `type: TLEntityType`: The type of the entity (e.g., bold, italic, code).
- `offset: int`: The starting position of the entity in the `text` (UTF-16 code units).
- `length: int`: The length of the formatted segment in the `text` (UTF-16 code units).
- `language: Optional[str]`: For `pre` (code block) entities, the specified language.
- `url: Optional[str]`: For `text_link` entities, the URL.
- `document_id: Optional[int]`: For `custom_emoji` entities, the ID of the custom emoji document.

To convert `RawEntity` objects into `TLRPC.MessageEntity` objects suitable for the Telegram API, call the `to_tlrpc_object()` method on each `RawEntity`.

## Supported Entity Types (TLEntityType)

The parser supports the following entity types:

- **BOLD** (`*bold*`)
- **ITALIC** (`_italic_`)
- **UNDERLINE** (`__underline__`)
- **STRIKETHROUGH** (`~strikethrough~`)
- **SPOILER** (`||spoiler||`)
- **CODE** (`inline code`)
- **PRE** (code block) - can include an optional language specifier.
- **TEXT_LINK** (`[link text](http://example.com)`)
- **CUSTOM_EMOJI** (`[alt text](document_id)`) - `alt text` becomes the content of the entity, `document_id` is the emoji's ID.

## Usage Example

This example demonstrates how to parse a Markdown string and send it as a formatted message.

```python
from markdown_utils import parse_markdown
from client_utils import send_message

try:
    # Parse markdown text
    markdown_text = "*Bold text* with _italic_ and `code`"
    parsed = parse_markdown(markdown_text)
    
    # Convert entities to TLRPC objects
    tlrpc_entities = [entity.to_tlrpc_object() for entity in parsed.entities]
    
    # Send formatted message
    send_message({
        "peer": chat_id,
        "message": parsed.text,
        "entities": tlrpc_entities
    })
    
except SyntaxError as e:
    log(f"Markdown parsing error: {e}")
```

## Important Notes

- **UTF-16 Offsets & Lengths**: The `offset` and `length` in `RawEntity` (and the resulting `TLRPC.MessageEntity`) are calculated based on UTF-16 code units, as required by the Telegram API. The parser handles this conversion automatically.

- **Error Handling**: If the Markdown syntax is incorrect (e.g., unclosed tags), `parse_markdown` will raise a `SyntaxError`. It's good practice to wrap the call in a `try-except` block.

- **Nesting**: Basic nesting of styles (e.g., bold inside italic) is generally supported, but complex or ambiguous nesting might lead to unexpected results.

- **Escaping**: Special Markdown characters (`*`, `_`, `~`, `|`, `` ` ``, `[`, `]`, `\`) can be escaped with a backslash (`\`) if you want them to appear as literal characters. For example, `\*not bold\*` will render as `*not bold*`.

- **Code Blocks**:
  - Inline code is surrounded by single backticks (`` ` ``).
  - Fenced code blocks are surrounded by triple backticks (` ``` `).
  - An optional language identifier can be placed immediately after the opening triple backticks (e.g., ```` ```python ````).

- **Custom Emoji**: The syntax `[alt text](document_id)` is used. The `alt text` (e.g., the emoji character itself) becomes the text segment covered by the `TLRPC.TL_messageEntityCustomEmoji` entity, and `document_id` is the ID of the custom emoji. You can obtain the emoji ID by sending the emoji to [@AdsMarkdownBot](https://t.me/AdsMarkdownBot) on Telegram.

This parser provides a robust way to include rich text formatting in messages sent by your plugins.


# Alert Dialog Builder

A Pythonic wrapper for creating and managing Telegram-style AlertDialogs.

The `AlertDialogBuilder` class, found in `alert.py`, provides a convenient way to construct and display various types of alert dialogs within your plugins. It wraps `org.telegram.ui.ActionBar.AlertDialog.Builder` and simplifies its usage from Python.

## Basic Usage

```python
from alert import AlertDialogBuilder
from client_utils import get_last_fragment

def show_simple_dialog():
    context = get_last_fragment().getParentActivity()
    
    builder = AlertDialogBuilder(context)
    builder.set_title("Hello")
    builder.set_message("This is a simple dialog")
    builder.set_positive_button("OK")
    builder.show()
```

## Dialog Types

`AlertDialogBuilder` supports different styles of dialogs, controlled by the `progress_style` parameter in its constructor:

- `AlertDialogBuilder.ALERT_TYPE_MESSAGE` (default): Standard message dialog.
- `AlertDialogBuilder.ALERT_TYPE_LOADING`: Dialog with a determinate horizontal progress bar. Use `builder.set_progress(value)` to update.
- `AlertDialogBuilder.ALERT_TYPE_SPINNER`: Dialog with an indeterminate spinner, often used for loading states.

```python
# Loading dialog with progress bar
builder = AlertDialogBuilder(context, AlertDialogBuilder.ALERT_TYPE_LOADING)
builder.set_title("Loading...")
builder.set_progress(50)  # 50% progress
builder.show()

# Spinner dialog
builder = AlertDialogBuilder(context, AlertDialogBuilder.ALERT_TYPE_SPINNER)
builder.set_title("Please wait...")
builder.show()
```

## Key Methods

### Initialization

```python
AlertDialogBuilder(context: Context, progress_style: int = ALERT_TYPE_MESSAGE, resources_provider: Optional[Theme.ResourcesProvider] = None)
```

### Content

```python
# Set dialog title
builder.set_title("Dialog Title")

# Set main message content
builder.set_message("This is the dialog message")

# Make message text clickable (e.g., for links)
builder.set_message_text_view_clickable(True)

# Set custom Android View as content
builder.set_view(custom_view, height=-2)

# Display list of items
items = ["Option 1", "Option 2", "Option 3"]
def on_item_click(dialog_builder, index):
    log(f"Selected item {index}: {items[index]}")
    dialog_builder.dismiss()

builder.set_items(items, on_item_click)
```

### Buttons

```python
def on_positive_click(dialog_builder, button_id):
    log("Positive button clicked")
    dialog_builder.dismiss()

def on_negative_click(dialog_builder, button_id):
    log("Negative button clicked")
    dialog_builder.dismiss()

builder.set_positive_button("OK", on_positive_click)
builder.set_negative_button("Cancel", on_negative_click)
builder.set_neutral_button("Maybe", on_neutral_click)

# Make negative button red
builder.make_button_red(AlertDialogBuilder.BUTTON_NEGATIVE)
```

Listeners receive the `AlertDialogBuilder` instance and a button identifier (`AlertDialogBuilder.BUTTON_POSITIVE`, etc.).

### Listeners

```python
def on_back_pressed(dialog_builder, button_id):
    log("Back button pressed")
    return True  # Consume the event

def on_dismiss(dialog_builder):
    log("Dialog dismissed")

def on_cancel(dialog_builder):
    log("Dialog cancelled")

builder.set_on_back_button_listener(on_back_pressed)
builder.set_on_dismiss_listener(on_dismiss)
builder.set_on_cancel_listener(on_cancel)
```

### Appearance & Behavior

```python
# Set top image/animation
builder.set_top_image(R.drawable.icon, background_color)
builder.set_top_drawable(drawable, background_color)
builder.set_top_animation(R.raw.animation, size=100, auto_repeat=True, background_color=0xFFFFFF)

# Control dimming and blur
builder.set_dim_enabled(True)
builder.set_blurred_background(True, blur_behind_if_possible=True)

# Set dialog button colors
builder.set_dialog_button_color_key(Theme.key_color_blue)

# Control cancelable behavior (best called after create() or show())
builder.set_cancelable(True)
builder.set_canceled_on_touch_outside(True)
```

### Lifecycle

```python
# Create dialog but don't show it
builder.create()

# Create (if not already) and show dialog
builder.show()

# Dismiss dialog if showing
builder.dismiss()

# Get underlying Java AlertDialog instance
dialog = builder.get_dialog()

# Get button view for custom styling (call after create() or show())
positive_button = builder.get_button(AlertDialogBuilder.BUTTON_POSITIVE)
```

### Progress

```python
# Set progress for ALERT_TYPE_LOADING dialogs (0-100)
builder.set_progress(75)
```

## Example: Dialog with Items

```python
from alert import AlertDialogBuilder
from client_utils import get_last_fragment

def show_options_dialog():
    context = get_last_fragment().getParentActivity()
    
    options = ["Edit", "Delete", "Share", "Copy Link"]
    
    def handle_option(builder, index):
        selected = options[index]
        log(f"User selected: {selected}")
        
        if selected == "Delete":
            # Show confirmation dialog
            show_confirmation_dialog()
        elif selected == "Edit":
            # Open edit dialog
            pass
        
        builder.dismiss()
    
    builder = AlertDialogBuilder(context)
    builder.set_title("Choose Action")
    builder.set_items(options, handle_option)
    builder.set_negative_button("Cancel")
    builder.show()
    
def show_confirmation_dialog():
    context = get_last_fragment().getParentActivity()
    
    def on_confirm(builder, button_id):
        log("Confirmed deletion")
        # Perform deletion
        builder.dismiss()
    
    builder = AlertDialogBuilder(context)
    builder.set_title("Confirm Deletion")
    builder.set_message("Are you sure you want to delete this item?")
    builder.set_positive_button("Delete", on_confirm)
    builder.set_negative_button("Cancel")
    builder.make_button_red(AlertDialogBuilder.BUTTON_POSITIVE)
    builder.show()
```

## Important Notes

- **Context**: Always provide a valid Android `Context` (usually an `Activity`) to the constructor. `get_last_fragment().getParentActivity()` is a common way to get this.

- **Listeners**: The listener callables you provide will receive the Python `AlertDialogBuilder` instance as their first argument, allowing you to interact with the dialog (e.g., `bld.dismiss()`) from within the callback.

- **Thread Safety**: Dialog manipulation (creating, showing, dismissing, updating content) should generally happen on the Android UI thread. Use `android_utils.run_on_ui_thread` if you're performing these actions from a background thread.

- **Error Handling**: The proxy listeners in `alert.py` include basic try-except blocks to log errors occurring within your Python callbacks, preventing crashes.


# Bulletin Helper

Легко отображайте различные типы уведомлений в нижней части экрана (Bulletins) в ваших плагинах.

Класс `BulletinHelper`, находящийся в `bulletin.py`, предоставляет набор статических методов для удобного отображения "Bulletin" уведомлений Telegram. Bulletins - это небольшие, ненавязчивые сообщения, которые обычно появляются в нижней части экрана и автоматически исчезают.

## Базовое использование

Большинство методов `BulletinHelper` являются методами класса и могут быть вызваны напрямую. Они часто принимают необязательный аргумент `fragment`; если он не предоставлен, помощник пытается использовать текущий активный фрагмент или глобальный контекст.

### UI Thread

Все методы `BulletinHelper.show_...` автоматически обеспечивают отображение bulletin в UI потоке Android, поэтому вам не нужно оборачивать эти вызовы в `run_on_ui_thread` самостоятельно.

## Типы Bulletin и методы

`BulletinHelper` оборачивает общие функции `org.telegram.ui.Components.BulletinFactory`.

### Стандартные Bulletins

#### `BulletinHelper.show_info(message: str, fragment: Optional[BaseFragment] = None)`
- Показывает bulletin с иконкой информации по умолчанию (например, `R.raw.info`)

#### `BulletinHelper.show_error(message: str, fragment: Optional[BaseFragment] = None)`
- Показывает bulletin с иконкой ошибки/предупреждения по умолчанию

#### `BulletinHelper.show_success(message: str, fragment: Optional[BaseFragment] = None)`
- Показывает bulletin с иконкой успеха/галочки по умолчанию

### Пользовательские простые Bulletins

#### `BulletinHelper.show_simple(text: str, icon_res_id: int, fragment: Optional[BaseFragment] = None)`
- Показывает однострочный bulletin с пользовательской иконкой Lottie анимации
- `icon_res_id`: ID ресурса Lottie анимации (например, `R_tg.raw.some_animation`)

#### `BulletinHelper.show_two_line(title: str, subtitle: str, icon_res_id: int, fragment: Optional[BaseFragment] = None)`
- Показывает двухстрочный bulletin с пользовательской иконкой, заголовком и подзаголовком

### Bulletins с действиями

#### `BulletinHelper.show_with_button(text: str, icon_res_id: int, button_text: str, on_click: Optional[Callable[[], None]], fragment: Optional[BaseFragment] = None, duration: int = BulletinHelper.DURATION_PROLONG)`
- Показывает bulletin с иконкой, текстом и кликабельной кнопкой
- `on_click`: Функция для выполнения при нажатии кнопки
- `duration`: Время отображения bulletin (например, `BulletinHelper.DURATION_SHORT`, `DURATION_LONG`, `DURATION_PROLONG`)

#### `BulletinHelper.show_undo(text: str, on_undo: Callable[[], None], on_action: Optional[Callable[[], None]] = None, subtitle: Optional[str] = None, fragment: Optional[BaseFragment] = None)`
- Показывает bulletin в стиле "Отменить"
- `on_undo`: Вызывается при нажатии кнопки "Отменить"
- `on_action`: Вызывается после задержки, если "Отменить" не было нажато (например, для подтверждения действия)

### Контекстные Bulletins (предопределенные)

#### `BulletinHelper.show_copied_to_clipboard(message: Optional[str] = None, fragment: Optional[BaseFragment] = None)`
- Показывает "Текст скопирован в буфер обмена" или пользовательское сообщение

#### `BulletinHelper.show_link_copied(is_private_link_info: bool = False, fragment: Optional[BaseFragment] = None)`
- Показывает bulletin "Ссылка скопирована", с вариантом для информации о приватной ссылке

#### `BulletinHelper.show_file_saved_to_gallery(is_video: bool = False, amount: int = 1, fragment: Optional[BaseFragment] = None)`
- Показывает "Фото/Видео сохранено в галерею" (или формы множественного числа)

#### `BulletinHelper.show_file_saved_to_downloads(file_type_enum_name: str = "UNKNOWN", amount: int = 1, fragment: Optional[BaseFragment] = None)`
- Показывает "Файл сохранен в загрузки" или аналогичное, на основе `BulletinFactory.FileType`
- `file_type_enum_name`: Строковое имя перечисления из `BulletinFactory.FileType` (например, `"PHOTO_TO_DOWNLOADS"`, `"GIF"`)

## Длительности

Класс `BulletinHelper` определяет константы для общих длительностей:

- `BulletinHelper.DURATION_SHORT` (1500 мс)
- `BulletinHelper.DURATION_LONG` (2750 мс)
- `BulletinHelper.DURATION_PROLONG` (5000 мс)

Эти константы можно использовать с методами вроде `show_with_button`.

## Поиск Lottie анимаций (R.raw...)

Lottie анимации, используемые для иконок bulletin, обычно хранятся как raw ресурсы в кодовой базе Telegram. Вы можете изучить исходный код Telegram (конкретно `TMessagesProj/src/main/res/raw/`) чтобы найти доступные анимации (например, `info.json`, `success.json`, `delete.json`). В Python к ним обращаются через `org.telegram.messenger.R.raw.animation_name` (например, `R_tg.raw.info`).


# Общие классы Telegram

Справочник по часто используемым классам исходного кода Telegram и их расположению для разработки плагинов.

## Ссылки на часто используемые классы Telegram

Рекомендуется иметь локальную копию исходников Telegram, открытую в Android Studio.

## LaunchActivity

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/LaunchActivity.java`

Здесь происходит инициализация приложения. Пользовательские ссылки также обрабатываются здесь.

## ProfileActivity

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/ProfileActivity.java`

Фрагмент профиля пользователя и канала.

## ChatActivity

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`

Отрисовка и функциональность чата.

## MessageObject

**Путь:** `TMessagesProj/src/main/java/org/telegram/messenger/MessageObject.java`

Обертка для `TLRPC.Message`.

## ChatMessageCell

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/Cells/ChatMessageCell.java`

Обрабатывает отрисовку сообщений.

## AndroidUtilities

**Путь:** `TMessagesProj/src/main/java/org/telegram/messenger/AndroidUtilities.java`

Содержит множество полезных утилит.

## MessagesController

**Путь:** `TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java`

Содержит все методы, связанные с управлением состоянием приложения и запросами Telegram.

## MessagesStorage

**Путь:** `TMessagesProj/src/main/java/org/telegram/messenger/MessagesStorage.java`

Управляет состоянием локальной базы данных. Вы можете использовать поле `database` для выполнения пользовательских SQLite запросов.

## SendMessagesHelper

**Путь:** `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

Содержит методы для отправки всех видов сообщений, включая файлы.

## BulletinFactory

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/Components/BulletinFactory.java`

Bulletins - это небольшие уведомления, отображаемые внизу экрана.

## AlertDialog

**Путь:** `TMessagesProj/src/main/java/org/telegram/ui/ActionBar/AlertDialog.java`

Диалоги предупреждений отображаются поверх текущего фрагмента. Они поддерживают пользовательские обработчики для кнопок действий.

## TLRPC (все модели запросов Telegram)

**Путь:** `TMessagesProj/src/main/java/org/telegram/tgnet`

Человекочитаемый список: [https://corefork.telegram.org/schema](https://corefork.telegram.org/schema) (не всегда актуален)
