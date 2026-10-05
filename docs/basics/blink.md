# Blinking an LED

Let's have a look at our ESP32 microcontroller:

<img width="785" alt="image" src="https://github.com/user-attachments/assets/a5708d32-d09b-4e50-b8bb-e7a262bb795e" />

As you can see, there are a lot of pins underneath it. These are called **GPIO pins** (General Purpose Input Output pins). These pins are the primary way in which the ESP32 control external hardware. Each pin can be turned HIGH voltage or LOW voltage.

What we are going to now is to make the GPIO22 pin HIGH and LOW continuously (like it's blinking), and connect an LED to that pin so it will turn ON and OFF. No need to worry about why we chose GPIO22 for now.

## Hardware required

Everything required for [Your First Project](/01-getting-started/02-first-project), plus:

- LED
- Jumper wires
- A 220-330 Ω resistor
- Breadboard (optional, but recommended)

## Create a new empty project

Create a new empty project using the `sample_project` template. We covered it in this chapter: [Your first project](/01-getting-started/02-first-project). But try to do it yourself without looking.

## Write the code

```c
// Include required headers
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

void app_main(void)
{
    // Clean up previous configuration
    gpio_reset_pin(GPIO22);

    // Configure the pin for voltage output
    gpio_set_direction(GPIO22, GPIO_MODE_OUTPUT);

    // Store the current state (0 = OFF, 1 = ON)
    uint8_t state = 0;

    ESP_LOGI("MAIN", "Starting LED...")

    // Infinite loop
    while (1) {
        // Set GPIO level (0 = OFF, 1 = ON)
        gpio_set_level(GPIO22, state);

        // Log the current state
        ESP_LOGI("MAIN", "LED State: %s", state ? "ON" : "OFF");

        // Toggle state for next iteration
        state = !state;

        // Stop execution for 1000 ms (1 second)
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

Here, `pdMS_TO_TICKS` is a function-like macro that converts milliseconds to ticks. `vTaskDelay` only accepts ticks.

## Improving the code

`GPIO_NUM22` and the `TAG` are used in multiple parts of the code. It's better to define them in the top of the file, rather than repeating it each time:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define BLINK_GPIO GPIO_NUM_22 // object-like macro to define gpio to be blinked

static const char *TAG = "MAIN"; // used for logging

void app_main(void)
{
    gpio_reset_pin(BLINK_GPIO);
    gpio_set_direction(BLINK_GPIO, GPIO_MODE_OUTPUT);

    uint8_t state = 0;

    ESP_LOGI(TAG, "Starting LED...")

    while (1) {
        gpio_set_level(BLINK_GPIO, state);
        ESP_LOGI(TAG, "LED State: %s", state ? "ON" : "OFF");

        state = !state;

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

## Build & flash

We covered this in previous chapters. Do you still remember it?

??? question "How to build the project"
    Run the `ESP-IDF: Build Your Project` command from the command palette.

??? question "How to flash the project"
    Refer the following section: [Your First Project - Flashing](/01-getting-started/02-first-project/#flashing)

## Connect your LED

Now, your GPIO22 pin must be turning ON and OFF repeatedly. But we can't see it yet. Time to set up the circuit:

- Connect the positive terminal of your LED to GPIO22 (labelled as `D22`).
- Connect the negative terminal of your LED to a 220 Ω to 330 Ω resistor (this is important; without a resistor, you might damage your microcontroller).
- Connect the other end of the resistor to the GND pin (ground pin) in the ESP32.

Now, you should see your LED blinking. Congrats!
