# Chaquopy 16.1 Documentation

## Table of Contents
1. [Introduction](#introduction)
2. [Examples](#examples)
3. [Gradle Plugin](#gradle-plugin)
    - [Project Setup](#project-setup)
    - [Development](#development)
    - [Packaging](#packaging)
    - [Python Standard Library](#python-standard-library)
    - [Android Studio Plugin](#android-studio-plugin)
4. [Java API](#java-api)
5. [Python API](#python-api)
    - [Data Types](#data-types)
    - [Import Hook](#import-hook)
    - [Inheriting Java Classes](#inheriting-java-classes)
6. [Cross-language Issues](#cross-language-issues)
7. [FAQ](#faq)

---

## Introduction

Chaquopy provides a friendly and efficient way to use Python in your Android apps. It integrates seamlessly with the standard Android build system and APIs.

To start using Chaquopy, see the [Examples](#examples), then follow the [Gradle Plugin](#gradle-plugin) setup instructions in your own app.

---

## Examples

For examples of how to use Chaquopy, see the following apps:

*   **[chaquopy-matplotlib](https://github.com/chaquo/chaquopy-matplotlib)**: A simple example of embedding a Python library in an Android app.
*   **[chaquopy-console](https://github.com/chaquo/chaquopy-console)**: A template for running text-based scripts.
*   **[Electron Cash](https://github.com/Electron-Cash/Electron-Cash/tree/master/android)** ([Google Play](https://play.google.com/store/apps/details?id=org.electroncash.wallet)): A full-size production app which uses Chaquopy to integrate a Python library with a modern Android user interface.
*   **[The demo app](https://github.com/chaquo/chaquopy/tree/master/demo)** ([Google Play](https://play.google.com/store/apps/details?id=com.chaquo.python.demo3)): Containing the SDK’s complete unit test suite.

---

## Gradle Plugin

Chaquopy is distributed as a plugin for Android’s Gradle-based build system. It can be used in an Android application module (`com.android.application`), or an Android library module (`com.android.library`).

> **Note:** The Chaquopy plugin can only be used in one module per app: either in the app module, or in exactly one library module.

### Project Setup

#### Gradle Plugin
In your project’s `settings.gradle` file, find the `pluginManagement` repositories list, and make sure it includes `mavenCentral()`.

In your **top-level** `build.gradle` file, set the Chaquopy version:

```kotlin
plugins {
    id("com.chaquo.python") version "16.1.0" apply false
}
```

Also check the Android Gradle plugin version; it should be between 7.0.x and 8.13.x.

Then apply the Chaquopy plugin in the **module-level** `build.gradle` file (usually in the `app` directory):

```kotlin
plugins {
    id("com.chaquo.python")
}
```

#### `minSdk`
Your project’s `minSdk` must be at least **24**.

#### `abiFilters`
The Python interpreter is a native component, so you must use the `abiFilters` setting to specify which ABIs you want the app to support. The currently available ABIs are:

*   `armeabi-v7a`: for older Android devices (Python 3.11 and older only).
*   `arm64-v8a`: for current Android devices, and emulators on Apple silicon.
*   `x86`: for older emulators (Python 3.11 and older only).
*   `x86_64`: for current emulators.

**Kotlin DSL:**
```kotlin
android {
    defaultConfig {
        ndk {
            // On Apple silicon, you can omit x86_64.
            abiFilters += listOf("arm64-v8a", "x86_64")
        }
    }
}
```

**Groovy DSL:**
```groovy
android {
    defaultConfig {
        ndk {
            // On Apple silicon, you can omit x86_64.
            abiFilters "arm64-v8a", "x86_64"
        }
    }
}
```

#### `chaquopy` block
All of the Chaquopy plugin’s settings are configured with the `chaquopy` block in your module-level `build.gradle` file.

```kotlin
chaquopy {
    defaultConfig { }
    productFlavors { }
    sourceSets { }
}
```

#### `buildPython`
Some features require Python 3.8 or later to be available on the build machine. By default, Chaquopy will try to find Python on the PATH. If this doesn’t work, set your Python command using the `buildPython` setting:

```kotlin
chaquopy {
    defaultConfig {
        buildPython("C:/path/to/python.exe")
        // Or with arguments:
        // buildPython("C:/path/to/py.exe", "-3.8")
    }
}
```

### Development

#### Startup
Before your app can run any Python code, you must call `Python.start()`.

**Method 1: Using PyApplication**
If the app always uses Python, use `PyApplication` in your `AndroidManifest.xml`:

```xml
<application
    android:name="com.chaquo.python.android.PyApplication"
    ... >
```

**Method 2: Manual Start**
If the app only sometimes uses Python:

```java
// "context" must be an Activity, Service or Application object.
if (! Python.isStarted()) {
    Python.start(new AndroidPlatform(context));
}
```

#### Python version
You can set your app’s Python version. The default is 3.8. Available versions: 3.8, 3.9, 3.10, 3.11, 3.12, 3.13.

```kotlin
chaquopy {
    defaultConfig {
        version = "3.8"
    }
}
```

#### Source code
By default, Chaquopy looks for Python source code in `src/main/python`. To include other directories:

```kotlin
chaquopy {
    sourceSets {
        getByName("main") {
            srcDir("some/other/dir")
        }
    }
}
```

#### Requirements
External Python packages may be built into the app using the `pip` block.

```kotlin
chaquopy {
    defaultConfig {
        pip {
            install("scipy")
            install("requests==2.24.0")
            install("MyPackage-1.2.3-py2.py3-none-any.whl") // Relative to project dir
            install("./MyPackage") // Directory containing setup.py
            install("-r", "requirements.txt")
            
            // Pass options
            options("--extra-index-url", "https://example.com/private/repository")
        }
    }
}
```
You can browse available native packages at [chaquo.com/pypi-13.1](https://chaquo.com/pypi-13.1/).

#### Static proxy generator
Allows a Python class to extend a Java class. *Requires `buildPython`.*

```kotlin
chaquopy {
    defaultConfig {
        staticProxy("module.one", "module.two")
    }
}
```

### Packaging

#### `extractPackages`
If certain packages need to exist as separate files (not loaded directly from APK), declare them here:

```kotlin
chaquopy {
    defaultConfig {
        extractPackages("package1", "package2.subpkg")
    }
}
```

#### Data files
Data files in source code or requirements are built into the app. Read them relative to `__file__`:

```python
from os.path import dirname, join
filename = join(dirname(__file__), "filename.txt")
```

#### Bytecode compilation
Enabled by default. Compilation to `.pyc` improves startup time. To disable:

```kotlin
chaquopy {
    defaultConfig {
        pyc {
            src = false
            pip = true
            stdlib = true
        }
    }
}
```

### Python Standard Library

#### Unsupported modules
*   `crypt`, `grp`, `nis`, `spwd` (Require missing OS features).
*   `curses`, `dbm.gnu`, `dbm.ndbm`, `readline`, `tkinter`, `turtle` (Require missing libraries).

#### `multiprocessing`
Android does not support System V IPC. Use `multiprocessing.dummy` instead.

#### `os`
Do not write to the current directory. Use `os.environ["HOME"]` (internal storage).

```python
import os
from os.path import join
filename = join(os.environ["HOME"], "filename.txt")
```

#### `sys`
`sys.stdout` and `sys.stderr` are redirected to Logcat (`python.stdout` / `python.stderr`). `sys.stdin` always returns EOF.

### Android Studio Plugin
You may install the "Python Community Edition" plugin for syntax highlighting, but ignore "No Python interpreter configured" warnings.

---

## Java API

The Java API allows Java code to interact with Python.

**Summary:**
1.  Call `Python.start()` (if not using `PyApplication`).
2.  Call `Python.getInstance()`.
3.  Call `getModule()` to get a `PyObject`.
4.  Use `PyObject` methods.

**Examples:**

*Modules and classes:*
```java
// Python: import zipfile
PyObject zipfile = py.getModule("zipfile");

// Python: zf = zipfile.ZipFile("example.zip")
PyObject zf = zipfile.callAttr("ZipFile", "example.zip");

// Python: zf.debug = 2
zf.put("debug", 2);

// Python: zf.write("filename.txt", compress_type=zipfile.ZIP_STORED)
zf.callAttr("write", "filename.txt", new Kwarg("compress_type", zipfile.get("ZIP_STORED")));
```

*Primitives:*
```java
// Python: sys.maxsize
sys.get("maxsize").toLong();

// Python: sys.version
sys.get("version").toString();
```

*Containers:*
```java
// Python: sys.version_info[0]
sys.get("version_info").asList().get(0).toInt();

// Python: os.environ["HELLO"] = "world"
os.get("environ").asMap().put(PyObject.fromJava("HELLO"), PyObject.fromJava("world"));
```

---

## Python API

The `java` module provides facilities to use Java classes and objects from Python code.

### Data Types

*   **Null:** Java `null` ↔ Python `None`.
*   **Booleans/Numbers:** Mapped to `bool`, `int`, `float`.
*   **Strings:** Java `String`/`char` ↔ Python `str`.
*   **Arrays:** Java arrays ↔ `jarray` objects (or Python sequences for creation).
*   **Objects:** All other objects ↔ `jclass`.

#### Primitives
If explicit types are needed for overload resolution:
`java.jboolean`, `java.jbyte`, `java.jshort`, `java.jint`, `java.jlong`, `java.jfloat`, `java.jdouble`, `java.jchar`.

Example:
```python
p.print(jint(42))      # Calls print(int)
p.print(jfloat(42.0))  # Calls print(float)
```

#### Classes
```python
from java import jclass
Calendar = jclass("java.util.Calendar")
c = Calendar.getInstance()
print(c.getTime())
```

#### Arrays
Created implicitly from Python sequences (except strings).
Explicit creation: `jarray(jint)([1, 2, 3])`.

Arrays support `[]` access, iteration, `len()`, and `bytes()` conversion.

#### Casting
Use `java.cast(cls, obj)` to view an object as a different type (useful for accessing interfaces or resolving overloads).

### Import Hook
Allows importing Java classes using Python syntax:
```python
from java.lang import System, Thread
from java.util import ArrayList
```
*Note: Wildcard imports (`*`) are not supported.*

### Inheriting Java Classes

#### Dynamic Proxy
Used to implement Java interfaces at runtime.
```python
from java import dynamic_proxy
from java.lang import Runnable

class R(dynamic_proxy(Runnable)):
    def run(self):
        print("Running")
```

#### Static Proxy
Used to extend Java classes or be referenced from Java. Requires `staticProxy` config in `build.gradle`.

```python
from java import static_proxy, method, jvoid, jint
from java.lang import Runnable

class C(static_proxy(Runnable)):
    @method(jvoid, [jint])
    def f(self, i):
        print(i)
```

### Exceptions
Java exceptions can be caught in Python:
```python
from java.lang import Integer, IllegalArgumentException
try:
    Integer.parseInt("abc")
except IllegalArgumentException as e:
    print(e)
```

### Multi-threading
The GIL is released when calling Java methods.
If you create threads using non-Java/non-Python means, use `java.detach()` before exit.

---

## Cross-language Issues

### Multi-threading
Chaquopy is thread-safe but limited by the CPython Global Interpreter Lock (GIL). Only one Python thread executes at a time.
*   `threading.main_thread` returns the thread where Python started.
*   Improve UI responsiveness by reducing `sys.setswitchinterval`.

### Memory Management
**Reference Cycles:** A cycle between a Python object and a Java object (where one holds a reference to the other) cannot be detected by either garbage collector. Use weak references or break cycles manually to avoid leaks.

---

## FAQ

### General
*   **Name:** "Chaquopy" comes from "cheechako" (newcomer).
*   **React Native/Flutter:** Yes, supported if you can edit `build.gradle` and call Java.
*   **iOS:** No. (See BeeWare).
*   **Size:** Reduce size by checking Python source, requirements, and using `abiFilters`.
*   **Obfuscation:** Code is compiled to `.pyc`. For more, use compiled Cython modules or copy/modify code in `build.gradle`.

### How do I...
*   **Read files:** Use path relative to `__file__`.
*   **Read External Storage:** Use Android's `ContentResolver` and `openInputStream` to pass bytes to Python.
*   **Write files:** Write to `os.environ["HOME"]`.
*   **Pass Images:** Encode as PNG/JPG byte array or pass raw arrays.

### Common Errors
*   **"Chaquopy cannot compile native code":** Package not pre-built. Check available versions.
*   **"No Python interpreter configured":** Harmless warning in Android Studio.
*   **"FileNotFoundError":** Check file paths.
*   **"ModuleNotFoundError":** Ensure package is listed in `pip` block.
*   **"No address associated with hostname":** Check INTERNET permission.