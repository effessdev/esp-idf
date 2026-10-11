# Mutexes

!!! warning "Draft"
    This chapter is not completed yet.

You might have heard this buzzword before. But it's not as complex as it seems. Let me show you exactly what they are by showing a specific situation.

## Let's create a function: `blink_led(int type)`

Let's create a function that blinks an LED. It takes a type argument. Type 1 means fast blink, and type 2 means slow blink.

Do not copy the function yet. It's missing headers.

```c
// Define macros for LED blink types
#define BLINK_TYPE_FAST 1
#define BLINK_TYPE_SLOW 2

#define LED_GPIO GPIO_NUM_22

void blink_led(int type)
{
    int interval_ms = 0;
    int total_blinks = 0;

    // Configure blink parameters depending on the requested type
    if (type == BLINK_TYPE_FAST)
    {
        // Type 1: Fast blink
        interval_ms = 250;
        total_blinks = 8;
    }
    else if (type == BLINK_TYPE_SLOW)
    {
        // Type 2: Slow blink
        interval_ms = 500;
        total_blinks = 4;
    }
    else
    {
        return;
    }

    // Perform the blinking sequence
    for (int i = 0; i < total_blinks; i++)
    {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(interval_ms));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(interval_ms));
    }
}
```

Now assume that this blink function is called by two tasks (Task A and Task B), running in parallel. What will happen?

## The problem

If Task B tries to blink the LED while Task A is blinking the LED, things will mix up and the LED will behave incorrectly.

Now, how do we make Task B wait until Task A stops? So only one task is blinking the LED (calling the `blink_led` function) at a time?

We can keep a global variable `is_blinking`, and make it `true` before starting to blink, and make it `false` once blinking finishes right?

And ensure `is_blinking` is `false` before starting to blink, or wait until it becomes `false` if it's not, right?

That variable is what we call a **mutex**. It's technically not a variable (an object), but we'll get to the details soon.

## Set up the tasks

Let's create Task A and Task B. They do certain kinds of work. But occasionally, they also blink the LED (calls the `blink_led()` function).

Delete everything in `main.c`, and define the tasks:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "esp_log.h"
#include "driver/gpio.h"

// Define macros for LED blink types
#define BLINK_TYPE_FAST 1
#define BLINK_TYPE_SLOW 2

// Define the GPIO pin used for the LED
#define LED_GPIO GPIO_NUM_22

// Handle to store our mutex (type: SemaphoreHandle_t)
SemaphoreHandle_t led_mutex = NULL;

void task_a(void *pvParameters)
{
    while (1)
    {
        ESP_LOGI("Task A", "Working (Fast Blink)...");
        // Call the blink_led function
        blink_led(BLINK_TYPE_FAST);
        ESP_LOGI("Task A", "Finished fast blink.");

        ESP_LOGI("Task A", "Doing other tasks.");
        // Mimick work with delay
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void task_b(void *pvParameters)
{
    while (1)
    {
        ESP_LOGI("Task B", "Working (Slow Blink)...");
        blink_led(BLINK_TYPE_SLOW);
        ESP_LOGI("Task B", "Finished slow blink.");
        
        ESP_LOGI("Task B", "Doing other tasks.")
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

## Write the main function

Now, create a mutex object and store it in the `led_mutex` handle. Also start the two tasks, and reset the `LED_GPIO`.

```c
// ... Existing code ...

void app_main(void)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    // Create the mutex
    led_mutex = xSemaphoreCreateMutex();

    if (led_mutex != NULL)
    {
        // Start both tasks with the same priority (5)
        xTaskCreate(task_a, "Task A", 2048, NULL, 5, NULL);
        xTaskCreate(task_b, "Task B", 2048, NULL, 5, NULL);
    }
    else // When mutex creation fails due to low memory
    {
        ESP_LOGE("APP_MAIN", "Failed to create mutex");
    }
}
```

## Function to blink the LED

Now, let's get to the actual part.

We use `xSemaphoreTake()` to mark the mutex as *taken*. Like this (do not copy it yet):

```c
xSemaphoreTake(led_mutex, portMAX_DELAY)
```

Here is the best part: **the `xSemaphoreTake` function doesn't return until it's able to take the mutex**. So it waits there indefinitly until the mutex becomes free so it can take it.

While waiting, it consumes zero CPU usage.

Once we are done with the blinking, we can use `xSemaphoreGive(led_mutex)` to mark the mutex as *free* again, so other tasks waiting for it to become free will be able to use it.

The second argument tells how long it has to wait. `portMAX_DELAY` is a macro that means "wait forever". So it won't stop after waiting for a bit.

Here is the `blink_led()` function:

```c
void blink_led(int type)
{
    int interval_ms = 0;
    int total_blinks = 0;

    if (type == BLINK_TYPE_FAST)
    {
        // Type 1: Fast blink
        interval_ms = 250;
        total_blinks = 8;
    }
    else if (type == BLINK_TYPE_SLOW)
    {
        // Type 2: Slow blink
        interval_ms = 500;
        total_blinks = 4;
    }
    else
    {
        return;
    }

    // Try to take the mutex.
    if (xSemaphoreTake(led_mutex, portMAX_DELAY) == pdTRUE)
    {
        // Perform the blinking sequence
        for (int i = 0; i < total_blinks; i++)
        {
            gpio_set_level(LED_GPIO, 1);
            vTaskDelay(pdMS_TO_TICKS(interval_ms));
            gpio_set_level(LED_GPIO, 0);
            vTaskDelay(pdMS_TO_TICKS(interval_ms));
        }

        // Give the mutex back so other tasks can take it
        xSemaphoreGive(led_mutex);
    }
}
```

Now, `blink_led()` function ensures that 2 blinks won't happen at the same time. You can now test your code on your ESP32. It should blink slowly and fast alternatively.

## Full final code

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "esp_log.h"
#include "driver/gpio.h"

// Define macros for LED blink types
#define BLINK_TYPE_FAST 1
#define BLINK_TYPE_SLOW 2

// Define the GPIO pin used for the LED
#define LED_GPIO GPIO_NUM_22

// Declare the mutex handle globally so both tasks can access it
SemaphoreHandle_t led_mutex = NULL;

// Function to blink the LED based on the specified type for 4 seconds
void blink_led(int type)
{
    int interval_ms = 0;
    int total_blinks = 0;

    if (type == BLINK_TYPE_FAST)
    {
        // Type 1: Fast blink
        interval_ms = 250;
        total_blinks = 8;
    }
    else if (type == BLINK_TYPE_SLOW)
    {
        // Type 2: Slow blink
        interval_ms = 500;
        total_blinks = 4;
    }
    else
    {
        return;
    }

    if (xSemaphoreTake(led_mutex, portMAX_DELAY) == pdTRUE)
    {
        // Perform the blinking sequence
        for (int i = 0; i < total_blinks; i++)
        {
            gpio_set_level(LED_GPIO, 1);
            vTaskDelay(pdMS_TO_TICKS(interval_ms));
            gpio_set_level(LED_GPIO, 0);
            vTaskDelay(pdMS_TO_TICKS(interval_ms));
        }

        // Give the mutex back so other tasks can take it
        xSemaphoreGive(led_mutex);
    }
}

void task_a(void *pvParameters)
{
    while (1)
    {
        ESP_LOGI("Task A", "Working (Fast Blink)...");
        // Call the blink_led function
        blink_led(BLINK_TYPE_FAST);
        ESP_LOGI("Task A", "Finished fast blink.");

        ESP_LOGI("Task A", "Doing other tasks.");
        // Mimick work with delay
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void task_b(void *pvParameters)
{
    while (1)
    {
        ESP_LOGI("Task B", "Working (Slow Blink)...");
        blink_led(BLINK_TYPE_SLOW);
        ESP_LOGI("Task B", "Finished slow blink.");
        
        ESP_LOGI("Task B", "Doing other tasks.")
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void app_main(void)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    // Create the mutex
    led_mutex = xSemaphoreCreateMutex();

    if (led_mutex != NULL)
    {
        // Start both tasks with the same priority (5)
        xTaskCreate(task_a, "Task A", 2048, NULL, 5, NULL);
        xTaskCreate(task_b, "Task B", 2048, NULL, 5, NULL);
    }
    else // When mutex creation fails due to low memory
    {
        ESP_LOGE("APP_MAIN", "Failed to create mutex");
    }
}
```
