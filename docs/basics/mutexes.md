# Mutexes

!!! warning "Draft"
    This chapter is not completed yet.

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

    // Try to take the mutex.
    // `portMAX_DELAY` is a macro that means "wait forever".
    // That means, `xSemaphoreTake()` will try to take the mutex indefinitely until it becomes available.
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
        // Call the blink_led function (mutex handling is now encapsulated inside)
        blink_led(BLINK_TYPE_FAST);
        ESP_LOGI("Task A", "Finished fast blink.");

        // Do whatever else task_a needs to do.
        // For now, let's mimic other work by delaying for a bit
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