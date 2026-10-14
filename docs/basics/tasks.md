# Tasks

In the previous project, we blinked an LED in GPIO22 in `app_main()` function. But what if we need to keep the LED blinking, while allowing `app_main()` to do other stuff?

Here is how we approach this:

- Create a function that blinks the LED indefinitely
- Start that function as a "Task" from `app_main()`, and let it run in parallel to `app_main()`

Now, `app_main()` can do whatever else. The function that runs in parallel is called a "Task". In this case, the function that blinks the LED.

## Task priority

This time, we are only using a single task, so task priority doesn't matter that much. But it matters when multiple tasks are running in parallel.

Task priority is a number assigned to a task that represents the priority of the task. Higher priority tasks block lower priority tasks. Lower priority tasks can only execute if higher priority tasks stop (temporarily or permanently). If both tasks have the same priority, they run in parallel. We'll learn more about these later in the course.

## Code

We have to first define the task (let's call it `blink_task`), and start the task from `app_main()` using the `xTaskCreate()` function:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_22

// Define the task
void blink_task(void *pvParameter) {
    while (1) {
        gpio_set_level(LED_GPIO, 1);
        ESP_LOGI("LED", "LED turned ON!");
        vTaskDelay(pdMS_TO_TICKS(1000));

        gpio_set_level(LED_GPIO, 0);
        ESP_LOGI("LED", "LED turned OFF!");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void) {
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    // Start the task
    xTaskCreate(
        blink_task,    /* The task function */
        "blink_task",  /* Name of the task (used for debugging) */
        2048,          /* RAM allocated to the task (in bytes) */
        NULL,          /* Pointer to the parameters passed to the task function (NULL since we aren't giving any parameters) */
        1,             /* Priority of the task (higher number = higher priority) */
        NULL           /* Task handle to track or control the task later (we aren't using this as of now, so NULL) */
    );

    // Now you can do anything else here.
    // blink_task() will run in parallel.
}
```

Now, you can build and flash the code to ensure it's working.

!!! tip "Tip"
    `app_main()` is also a task. It just starts automatically. The priority of `app_main()` task is 1. The lowest priority value that can be given to a task is 0.

## Challege (optional)

Connect a new LED to GPIO23 (don't forget the resistor), and create a new task `blink_task_fast()` to blink that second LED faster than the first LED. Start both tasks from `app_main()` (one after the other) with the same priority so that both LEDs are blinking but in different speeds.
