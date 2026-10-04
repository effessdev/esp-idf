# Installation

We have to install 2 things:

1. **ESP-IDF:** The framework we use for programming our ESP32.
2. **ESP-IDF VS Code Extension:** To interact with ESP-IDF.

## Install ESP-IDF

Espressif Systems provides a graphical tool called **EIM (ESP-IDF Installation Manager)** to install ESP-IDF. We have to install EIM first.

### Install EIM

Click the link below to go to the official page to download EIM:

<https://dl.espressif.com/dl/eim/>

Make sure you are in the "Online Installer" tab. The exact file to download depends on your system:

- Windows: Download `eim-gui-windows-x64.exe`. Run this installer to install EIM.
- Linux x64 (Ubuntu): Download and install the `.deb` package (`eim-gui-linux-x64.deb`).

### Install ESP-IDF using EIM

Now that we have installed EIM, let's install ESP-IDF using it.

1. Open EIM.
2. Under "New Installation" click "Start Installation".
3. Under "Easy Installation", click "Start Easy Installation" to install the latest stable version of ESP-IDF with default settings.
4. If there are no problems, you will see the "Ready to Install" page. Click "Start Installation".

## Install ESP-IDF VS Code Extension

We use this extension as a high-level wrapper for ESP-IDF. Most times, we do not use ESP-IDF directly. For example, if we need to compile our source code, we ask the extension to do it, which uses the ESP-IDF we just installed internally to to compile the source code.

Install the extension named "ESP-IDF" by "Espressif Systems" in VS Code.

After installing, restart VS Code. You might get this notification:

> _No standard ESP-IDF project was found in this workspace. Do you want to activate the ESP-IDF extension anyway?_

Click "Activate Anyway". This will activate the extension.

### Verify installation

Use the shortcut `Ctrl + Shift + P` to open the **command palette** (remember this shortcut, we are going to use it a lot). Inside the command palette, search `ESP-IDF`. You will see many entries which start with `ESP-IDF:`. Those commands are provided my the ESP-IDF extension. These commands are what we use for almost everything.

## Test your knowledge

??? question "How do we open the command palette?"
    Using the shortcut `Ctrl + Shift + P`.
