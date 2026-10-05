# Your first project

Hardware required:

- An ESP32 microcontroller.
- A suitable cable to connect it to your PC.

## Create a new empty project

We are going to use the `ESP-IDF: New Project` command from the command palette to create the project.

- Open the command palette (I hope you remember the shortcut)
- Search for and click on `ESP-IDF: New Project`, and wait.
- Select your ESP-IDF version (you might see only one version since we only installed one), and wait.
- A new tab will pop up. Inside that tab, under `ESP-IDF Templates`, select the `sample_project` template and click the "Create Project" button.

The tab will refresh, and you will see a form to fill in your project details. Fill in the details:

- **Project name:** Your project name.
- **Project directory:** Your project directory.
- **ESP-IDF target:** esp32.
- **ESP-IDF board:** Custom board.
- **Serial port:** Detect.
- **OpenOCD configuration files:** Keep the default value.
- **ESP-IDF component directory:** Keep the input empty.

Click "Create Project" and wait again. After creation, in the new tab that pops up, click "Open Project". This will open your brand new ESP-IDF project in a fresh VS Code window.

!!! note "If you are using Clangd"
    If you are using Clangd instead of the Microsoft C/C++ extension, run `ESP-IDF: Configure project for ESP-Clang` from the VS Code command palette to make sure Clangd IntelliSense works correctly. Also, make sure both of them aren't activated at the same time, as they can interfere with each other.

## Configure your project

You need to configure your project whenever you:

- Start a new project (like we did now)
- Add, remove, or rename source code files
- And so on.

But it's safe to do again and again. Since we just created a new project, let's run it by clicking the `ESP-IDF: Run idf.py reconfigure Task` command from the command palette.

This will take some time. While it's working, let's learn what it does:

- Generates (or regenerates) the build files in the `build` folder
- Generates `compile_commands.json`, which is essential for Microsoft C/C++ Extension or Clangd to provide IntelliSense
- And a lot more (just know these two for now)

## Let's write some code

If you look at `main.c` right now, you will see an empty main function (`app_main`):

```c
#include <stdio.h>

void app_main(void)
{

}
```

Let's log "Hello world!" inside it.

### Hello, world!

We use a function called `ESP_LOGI()` to log "Hello world!" (`I` stands for "info"). Include the `esp_log.h` header to use that function. The function takes two arguments: a `TAG`, and the actual string to log.

```c
#include "esp_log.h"

void app_main(void) {
  ESP_LOGI("MAIN", "Hello world!");
}
```

This code will log something like this:

```plaintext
I (312) MAIN: Hello world!
```

Here,

- `I`: Log level indicator (Info).
- `(312)`: Timestamp in milliseconds since boot (this exact number will vary).
- `MAIN`: Tag name (useful to determine the source of the LOG message when we have a lot of them).
- `Hello world!`: The actual content.

## Building (compiling) the project

Open the command palette (`Ctrl + Shift + P`) and run

```plaintext
ESP-IDF: Build Your Project
```

This will also take some time. Be patient.

## Flashing your project {#flashing}

Flashing is like uploading the compiled code to your ESP32. You need to physically connect the ESP32 to your computer. After doing that, run the following command from the command palette:

```plaintext
ESP-IDF: Flash (UART) Your Project
```

It probably won't work out of the box. The fix differs depending on your platform.

### If you are using Windows

If you are on Windows, flashing might show this error:

```plaintext
A fatal error occurred: Could not connect to an Espressif device on any of the 1 available serial ports.
```

If that happens, the next time you try, press and hold the BOOT button on your board as soon as you see `Connecting...`. After some time, the `Connecting...` will stop and you will see several different outputs. Stop holding the BOOT button when you see these somewhere:

```plaintext
Uploading stub flasher...
Running stub flasher...
Stub flasher running.
```

It might require some trial and error to find out how long you will have to hold the BOOT button.

### If you are using Linux (e.g., Ubuntu)

You might get this error while flashing:

```plaintext
A fatal error occurred: Could not open /dev/ttyUSB0, the port is busy or doesn't exist.
([Errno 13] could not open port /dev/ttyUSB0: [Errno 13] Permission denied: '/dev/ttyUSB0')

Hint: Try to add the user to the dialout or uucp group.
```

If that happens, run this command, log out, log back in (or reboot), and try flashing again:

```bash
sudo usermod -aG dialout $USER
```

You have to do it only once per system. Subsequent flashing will work without running this command.

## Monitoring the device

You should keep your ESP32 connected to your computer so that it receives power and is running. Let's check whether the "Hello world!" got logged.

At the bottom of VS Code, you will see several icons. Find the icon that looks like a monitor and hover over it. If you see "Monitor Device" when you hover, you found the correct icon.

Click on that icon. You will see several logs popping up. These are sent internally. But among those log messages, you will find our Hello world:

```plaintext
I (258) main_task: Started on CPU0
I (258) main_task: Calling app_main()
I (258) MAIN: Hello world!   <--------------- this
I (258) main_task: Returned from app_main()
```

You might see different numbers than 258. That's not a problem.

So, congratulations! You just set up your computer for ESP-IDF, created a new project, wrote some code, compiled, flashed, and monitored the output!

## Tips

### Adding the VS Code Configuration Folder

If you accidentally deleted/edited the `.vscode` folder, you can regenerate it using the `ESP-IDF: Add VS Code Configuration Folder` command from the command palette.

## Test Your Knowledge

Test your knowledge by answering the following questions:

- How do we create a new ESP-IDF project? Which template should we use for an empty project?
- When do we need to configure our project? How can we do it? Is it safe to do it repeatedly?
- What does the "I" in `ESP_LOGI` stand for?
- How do we build an ESP-IDF project?
- How do we flash an ESP-IDF project to the ESP32?
- What is monitoring? How can we do it?
