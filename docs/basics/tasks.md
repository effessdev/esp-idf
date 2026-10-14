# Tasks

In the previous project, we blinked an LED in GPIO22 in `app_main()` function. But what if we need to keep the LED blinking, while allowing `app_main()` to do other stuff?

Here is how we approach this:

- Create a function that blinks the LED indefinitely
- Start that function as a "Task" from `app_main()`, and let it run in parallel to `app_main()`

Now, `app_main()` can do whatever else. The function that runs in parallel is called a "Task". In this case, the function that blinks the LED.

## Where do tasks run?

ESP32s have two processor cores (Core 0 and Core 1). We can configure a task to run on either one of them.

## Task priority

Task priority is a number assigned to a task that represents the priority of the task. Let's consider a situation:

- Task A and Task B are running on the same processor core
- Task A has priority = 5, while Task B has priority = 6

In this case, only Task B will block the execution of Task A. Only Task B will run.

Task A can only run if:

- Task B temporarily stops itself (using `vTaskDelay()` or similar)
- Task B permanently stops itself (delete itself)
- Task B get stopped by another task

If both tasks have the same priority, they run in parallel.

## Code

We have to first define the task (let's call it `blink_task`), and start the task from `app_main()`:

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
        blink_task,    /* Function pointer to the task */
        "blink_task",  /* Descriptive name for debugging */
        2048,          /* Stack size allocated for the task in bytes */
        NULL,          /* Pointer to input parameters passed to the task (NULL since we aren't giving any input) */
        1,             /* Priority of the task (higher number = higher priority) */
        NULL,          /* Task handle to track or control the task later (we aren't using this as of now) */
        1              /* Core ID (0 or 1) */
    );

    // Now you can do anything else here.
    // `blink_task` will run in parallel.
}
```

If you don't give the Code ID argument, the task will be free to float between whichever CPU cores available (Core 0 or Core 1).

## Misconceptions

I said that if both Task A and Task B have the same priority and are pinned to the same CPU core, they run in parallel, right? But in reality, FreeRTOS will share the core between them using a method called "time-slicing". You can learn more about it online.

Also, `app_main()` is also a task. It just starts automatically.
