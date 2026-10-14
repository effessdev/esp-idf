# Blinking an LED

Let's have a look at our ESP32 microcontroller:

<img width="785" alt="image" src="https://github.com/user-attachments/assets/a5708d32-d09b-4e50-b8bb-e7a262bb795e" />

As you can see, there are a lot of pins underneath it. These are called **GPIO pins** (General Purpose Input Output pins). These pins are the primary way in which the ESP32 control external hardware. Each pin can be turned HIGH voltage or LOW voltage.

!!! warning "Note"
    All of those pins are not GPIO pins, but the majority are. You will discover other pins (like `GND`) later in the course.

What we are going to now is to make the GPIO22 pin HIGH and LOW continuously (like it's blinking), and connect an LED to that pin so it will turn ON and OFF. No need to worry about why we chose GPIO22 for now.

## Create a new empty project

Create a new empty project using the `sample_project` template. We covered it in this chapter: [Your First Project](../getting-started/first-project.md). Name it `led_blink`.

## Initialize Git

Run `git init` to initialize the repository. Commit the changes with the commit message "initial commit" and optionally push to GitHub.

## Write the code

The code is self-explanatory:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

void app_main(void)
{
    // Clean up previous configuration in GPIO22
    gpio_reset_pin(GPIO_NUM_22);

    // Configure GPIO22 for voltage output
    gpio_set_direction(GPIO_NUM_22, GPIO_MODE_OUTPUT);

    ESP_LOGI("MAIN", "Starting LED...");

    while (1) {
        // Set output voltage (1 means HIGH voltage)
        gpio_set_level(GPIO_NUM_22, 1);
        ESP_LOGI("MAIN", "LED State: ON");

        // Wait 1000 milliseconds (1 second)
        // pdMS_TO_TICKS() convert milliseconds to ticks (more on that later)
        vTaskDelay(pdMS_TO_TICKS(1000));

        // 0 means LOW voltage
        gpio_set_level(GPIO_NUM_22, 0);
        ESP_LOGI("MAIN", "LED State: OFF");

        // Wait for 1 second (again)
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Here, `pdMS_TO_TICKS` is a function-like macro that converts milliseconds to ticks. `vTaskDelay` only accepts ticks. If you do not know about macros, learn it right now by asking an AI. It's something that we use regularly.

You can commit the changes to your repository now.

## Improving the code

`GPIO_NUM22` and the `TAG` are used in multiple parts of the code. It's better to define them in the top of the file, rather than repeating each time:

```c
#include "driver/gpio.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <stdio.h>

// object-like macro to define the GPIO to be blinked
#define BLINK_GPIO GPIO_NUM_22

// used for logging
static const char *TAG = "MAIN";

void app_main(void) {
  gpio_reset_pin(BLINK_GPIO);
  gpio_set_direction(BLINK_GPIO, GPIO_MODE_OUTPUT);

  ESP_LOGI(TAG, "Starting LED...");

  while (1) {
    gpio_set_level(BLINK_GPIO, 1);
    ESP_LOGI(TAG, "LED State: ON");
    vTaskDelay(pdMS_TO_TICKS(1000));

    gpio_set_level(BLINK_GPIO, 0);
    ESP_LOGI(TAG, "LED State: OFF");
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
```

Commit these changes with a proper commit message.

## Build & flash

We covered this in previous chapters. Do you still remember it?

??? question "How to build the project"
    Run the `ESP-IDF: Build Your Project` command from the command palette.

??? question "How to flash the project"
    Refer to the following section: [Your First Project - Flashing](../getting-started/first-project.md/#flashing)

## Connect your LED

Now, your GPIO22 pin must be turning ON and OFF repeatedly. But we can't see it yet. Time to set up the circuit:

- Connect the positive terminal of your LED to GPIO22 (labelled as `D22`).
- Connect the negative terminal of your LED to a 220 Ω to 330 Ω resistor (this is important; without a resistor, you might damage your microcontroller).
- Connect the other end of the resistor to the GND pin (ground pin) in the ESP32.

Now, you should see your LED blinking. Congrats!
