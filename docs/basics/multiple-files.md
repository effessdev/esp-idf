# Using multiple files

!!! warning "Draft"
    This chapter is not complete yet.

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
#include "led.h"
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

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "led.h"
#include <stdio.h>

void app_main(void) {
  led_init();

  while (1) {
    led_toggle();
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
#include "led.h"
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
