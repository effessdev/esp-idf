# Tasks

Let's upgrade our blink LED code. Instead of blinking the LED directly inside `app_main()`, let's create a function `blink_led()` and call it from `app_main()`:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_22

// More on `void *pvParameter` later (hang tight)
void blink_led(void *pvParameter) {
    while (1) {
        gpio_set_level(LED_GPIO, 1);
        ESP_LOGI("LED", "LED State: ON");
        vTaskDelay(pdMS_TO_TICKS(1000));

        gpio_set_level(LED_GPIO, 0);
        ESP_LOGI("LED", "LED State: OFF");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void) {
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    blink_led();
}
```

Now, imagine we need to log "LED started blinking" after calling `blink_led()`. But this is not possible because `blink_led()` runs an infinite loop and doesn't return.

Even if we add `ESP_LOGI("LED", "LED started blinking")` after `blink_led()` call, it won't reach there.

## The fix

Instead of calling `blink_led()` directly, what if we let it run parallel to `app_main()`? `app_main()` only need to *start* `blink_led()`. It will run on it's own from then, and `app_main()` can do other stuff.

To do this, we create a **Task** using the `blink_led()` function. Once we create a task, it starts running parallel to `app_main()`.

We can use the `xTaskCreate()` function to create tasks. Update `app_main()` like this:

```c
void app_main(void) {
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    // Create the task
    xTaskCreate(
        blink_led,     /* The task function */
        "blink_task",  /* Name of the task (used for debugging) */
        2048,          /* RAM allocated to the task (in bytes) */
        NULL,          /* Pointer to the parameters passed to the task function blink_led (NULL since we aren't giving any parameters) */
        1,             /* Priority of the task (we'll discuss this later) */
        NULL           /* Task handle to track or control the task later (we aren't using this as of now, so NULL) */
    );

    // Now you can do anything else here.
    // Let's log something!
    ESP_LOGI("LED", "LED started blinking")
}
```

Now you can test your code & ensure it's working.

## A few improvements

Since `blink_led()` is a task, it's better to rename it to `blink_led_task()` or `blink_task()`. But it's optional.

## Task priority

In bigger firmware, we might have multiple tasks running in parallel. In those cases, task priority matters a lot.

Here are the key points:

- Task priority is a number assigned to a task that represents the priority of the task.
- Higher priority tasks block lower priority tasks. Lower priority tasks can only execute if higher priority tasks stop (temporarily or permanently).
- If both tasks have the same priority, they run in parallel.

We will be creating programs with multiple tasks later in the course!
