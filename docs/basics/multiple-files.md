# Structuring your project

This is our current project structure (I skipped some files to make it smaller):

```
.
├── main
│   ├── CMakeLists.txt
│   └── main.c
├── .gitignore
└── CMakeLists.txt
```

When we built our `led_blink` project, we put all the code in `main.c`. But when the project grows, it would be very hard to maintain.

So, what we are going to now, is to define an `init_led` function and a `toggle_led` function in a separate file `main/led.c`, and use them in `main.c`. But doing this, we can keep `main.c` shorter & cleaner.

The new project layout will look like this:

```
.
├── main
│   ├── CMakeLists.txt
│   ├── led.c          <-- New file added
│   └── main.c
├── .gitignore
└── CMakeLists.txt
```

## Step 1: Create `main/led.c`

Paste the following code in `led.c` file. I am skipping the explanations since we already know this.

```c
#include "driver/gpio.h"
#include <stdbool.h>

#define LED_GPIO_NUM GPIO_NUM_22

// Keep hardware state internal to this file
static bool s_led_on = false;

void led_init(void) {
  gpio_config_t io_conf = {
      .pin_bit_mask = (1ULL << LED_GPIO_NUM),
      .mode = GPIO_MODE_OUTPUT,
      .pull_up_en = GPIO_PULLUP_DISABLE,
      .pull_down_en = GPIO_PULLDOWN_DISABLE,
      .intr_type = GPIO_INTR_DISABLE,
  };
  gpio_config(&io_conf);

  s_led_on = false;
  gpio_set_level(LED_GPIO_NUM, 0);
}

void led_toggle(void) {
  s_led_on = !s_led_on;
  gpio_set_level(LED_GPIO_NUM, s_led_on ? 1 : 0);
}
```

## Step 2: Add `led.c` to `main/CMakeLists.txt`

Your current `main/CMakeLists.txt` only contains `main.c`:

```cmake
idf_component_register(SRCS "main.c"
                    INCLUDE_DIRS ".")
```

Add `led.c` in `main/CMakeLists.txt` (not the root `CMakeLists.txt`, the one in the `main` folder):

```cmake
idf_component_register(SRCS "main.c" "led.c"
                    INCLUDE_DIRS ".")
```

We'll later cover what `idf_component_register` means. For now, let's just get this working.

## Step 3: Add `PRIV_REQUIRES` to `main/CMakeLists.txt`

Our `led.c` uses `#include "driver/gpio.h"`. This requires a component named `esp_driver_gpio`. We will cover what components are later in the course. For now, let's add it:

```cmake
idf_component_register(SRCS "main.c" "led.c"
                    INCLUDE_DIRS "."
                    PRIV_REQUIRES esp_driver_gpio)
```

If we are using `#include "driver/gpio.h"` in `main.c`, this isn't required. We are required to manually add the components if we use them in other files.

## Step 4: Update `main.c`

Now that `led.c` has been added, you can call the functions in `main.c`:

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <stdio.h>

void app_main(void) {
  led_init(); // from led.c

  while (1) {
    led_toggle(); // from led.c
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

## Step 5: Add declarations in `main.c`

The compiler compiles each C file separately, and later links them through a process called `linking`.

Our `main.c` calls `led_init()` and `led_toggle()`. But `main.c` has no access to `led_init()` and `led_toggle()` yet, so it doesn't know how to *call* those functions (which arguments to pass, what's the return type, etc.). For this, you have to *declare* the functions before calling them on top of the main file, like this:

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <stdio.h>

void led_init(void);    // Declaration for led_init()
void led_toggle(void);  // Declaration for led_toggle()

void app_main(void) {
  led_init();

  while (1) {
    led_toggle();
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

Now, when the compiler compiles this file, it leaves placeholders for the address of the functions, which the linker later replace with the actual address. The compiler was able to compile the *calling* part because we provided the declarations. If we didn't do that, the compiler would not know how to call those functions (whether arguments should be given, what type of arguments, where should the return value go, etc.).

Now, you can test the code.

## Improving the code

Instead of adding the declarations in `main.c`, we use something called **header files**. They have the `.h` file extension.

Remove the declarations from `main.c`, and move them to a new file `main/led.h`. You can add comments on top of the declarations like this:

```c
// Initialize the LED GPIO. Must be called once before led_toggle().
void led_init(void);

// Toggle the LED state (on -> off, off -> on).
void led_toggle(void);
```

To include these declarations, we use `#include "led.h"`. Update `main.c` like this:

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <stdio.h>
#include "led.h" // New header included

void app_main(void) {
  led_init();

  while (1) {
    led_toggle();
    vTaskDelay(pdMS_TO_TICKS(500));
  }
}
```

### How `#include` works

Before compiling the code, there is another step called **preprocessing**. In this step, `#include "led.h"` is replaced with the contents of `led.h`.

So, this is functionally similar to putting the declarations in `main.c` itself. But by separating declarations into a separate file, we can keep `main.c` cleaner.

### #pragma once

To ensure that no header get included multiple times (even if you do), add `#pragma once` on the very top of **all** header files. This is something you should remember every time you create a header file.

The updated header file would look like this:

```c
#pragma once

// Initialize the LED GPIO. Must be called once before led_toggle().
void led_init(void);

// Toggle the LED state (on -> off, off -> on).
void led_toggle(void);
```

### FAQs

??? question "Can we do `#include functions.c` in `main.c`?"
    Yes, and it works in this case. But it's considered as a bad practice. If we do that, the final file would contain the contents of both `main.c` and `functions.c`. Even if we only changed `functions.c`, the entire file will need to be recompiled, which is slow when you have many large files.

## For even bigger projects

!!! warning "The following section was made in a hurry 🏃"
    Because I didn't get enough time (sorry 😢), I wasn't able to complete the following section up to the standards of the rest of the course. But including it here anyway since I wasn't comfortable with completely omitting it. If you want to learn about ESP-IDF components in detail, please check out [this page from the official documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/build-system.html), but **only after reading the following section** (the official docs are not as good as my course lol but it's wayy more detailed).

As our project grows, we can organize related files into separate _components_ instead of keeping everything in the `main` folder.

For example, the project structure could look like this:

```text
.
├── main
│   ├── CMakeLists.txt
│   └── main.c
├── components
│   └── led
│       ├── CMakeLists.txt
│       ├── led.c
│       └── include
│           └── led.h
├── .gitignore
└── CMakeLists.txt
```

Each component has its own `CMakeLists.txt`, which defines its source files and other stuff.

Note that we put the header file in the `include` directory instead of directly along with `led.c` to make the component even cleaner.

### components/led/CMakeLists.txt

Since `led` is a separate component, we should specify which other components it depends on in `components/led/CMakeLists.txt`. If our `led` component uses `#include "driver/gpio.h"`, you should add the `esp_driver_gpio` component using `PRIV_REQUIRES` (or `REQUIRES` in some cases). We don't have to do this for `main/main.c` (only for other files).

```cmake
idf_component_register(
    SRCS "led.c"
    PRIV_REQUIRES esp_driver_gpio
)
```

So, before including any headers, search on Google which component we have to add in `PRIV_REQUIRES` to include that specific header in your specific ESP-IDF version (in this course, version 6).

We put the header file in the `include` folder, right? So, we have to add `INCLUDE_DIRS "include"` too. This makes it equivalent to putting the header directly in the component (next to `led.c`). Update it like this:

```cmake
idf_component_register(
    SRCS "led.c"
    INCLUDE_DIRS "include"
    PRIV_REQUIRES esp_driver_gpio
)
```

### main/CMakeLists.txt

Fun fact: our `main` is also a component! Since it depends on the `led` component, add `PRIV_REQUIRES led`:

```cmake
idf_component_register(
    SRCS "main.c"
    INCLUDE_DIRS "."
    PRIV_REQUIRES led
)
```

ESP-IDF automatically discovers components inside the `components` directory (that means we don't have to tell "I put my components in `components` folder"). This structure keeps code organized, makes components reusable, and helps us manage dependencies as the project grows.

---

<p style="text-align: center;">
  <a href="/esp-idf/thanks"><strong>Next chapter: Thanks 😊</strong></a>
</p>
