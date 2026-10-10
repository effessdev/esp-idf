# Timers

!!! warning "Draft"
    This chapter is not complete yet.

In the previous chapter, we learned how to use hardware interrupts to respond to a button press. But what if we want to perform an action **after a delay** when the button is pressed?

For example, imagine pressing a button and having an LED flash one second later.

We could use `vTaskDelay()` to wait for one second before turning the LED on. But if we do a second press and the ISR adds that event to the queue while it's waiting, that queue item isn't consumed by the button task until the first flash is complete.

So, the second flash happens 1 second *after* the first flash, not after 1 second from the second **click**.

To fix this, we use a **timer**.

## Key concepts

### What is a timer?

A timer allows us to schedule an action to happen (or a function to be called) after a specified amount of time.

What we are going to do is:

1. We press the button and the ISR handler adds a press event to the queue
2. The button task consumes the event from the queue, and schedules a `flash_led` function to be called after 1 second using a timer

Now, the job of the button task is complete! It can do anything else, the timer will call the `flash_led` function after 1 second.

### Callback function

Here, we are giving the flash function to the timer and telling it to call it after 1 second. These kinds of functions are called **callback functions**.

They aren't a different type of function. They are called so just because of it's purpose. It just like any other C function.

### One-shot and periodic timers

There are two common types of timers:

- **One-shot timer:** Invokes its callback once after the specified interval and then stops.
- **Periodic timer:** Invokes its callback repeatedly at the specified interval until it is stopped.

For our project, we will use a one-shot timer because we want the LED to flash once after a button press.

### The `esp_timer` API

ESP-IDF provides the `esp_timer` API for creating software timers.

Here are some of it's important functions. You don't have to memorize these. The names are self-explanatory.

| Function | Purpose |
|---|---|
| `esp_timer_create()` | Create a timer |
| `esp_timer_start_once()` | Start a one-shot timer |
| `esp_timer_start_periodic()` | Start a periodic timer |
| `esp_timer_stop()` | Stop a running timer |
| `esp_timer_restart()` | Restart an active timer with a new timeout |
| `esp_timer_delete()` | Delete a timer |

We will use `esp_timer_create()` and `esp_timer_start_once()` in this project.

### Are timers reliable?

Software timers do not guarantee perfectly exact callback execution times. System load and callback scheduling can introduce delays.

## Set up the project

Create a new empty project using the `sample_project` template, as covered in [Your First Project](../getting-started/first-project.md). Name it `led_timer`.

## Setup the circuit

We will use GPIO22 for the LED and GPIO23 for the button, just like in our previous chapters. The circuit is also same as before:

LED:

- Connect the positive terminal of the LED to GPIO22.
- Connect the negative terminal to a 220–330 Ω resistor.
- Connect the other end of the resistor to GND.

Button:

- Connect one end of the jumper-wire to GPIO23. We can touch it to GND for a button press.

Optionally, you can use an actual button.

## What we are going to do

1. The button task waits for press events in the queue
2. When an event enters the queue, the button task schedules `flash_led` for 1 second later
3. After 1 second, the timer calls `flash_led`.
4. `flash_led` turns the LED on, and schedules **another timer** to turn the LED off after a short amount of time. A new function `led_off_callback` is given to this timer.
5. After than short amount of time, the LED becomes OFF.

## Write the code

Replace the contents of `main.c` with the following code:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"
#include "esp_timer.h"

#define LED_PIN             GPIO_NUM_22
#define BUTTON_PIN          GPIO_NUM_23

#define FLASH_DELAY_MS      1000  // How long the LED waits before flashing
#define FLASH_DURATION_MS   100   // How long the LED be ON in a single flash

// Queue to send button events from ISR to the task
static QueueHandle_t gpio_evt_queue = NULL;

// Timer handles
static esp_timer_handle_t flash_timer;
static esp_timer_handle_t led_off_timer;

// ISR keeps work minimal (add the event to the queue)
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    uint32_t gpio_num = (uint32_t)arg;
    BaseType_t higher_priority_task_woken = pdFALSE;

    xQueueSendFromISR(
        gpio_evt_queue,
        &gpio_num,
        &higher_priority_task_woken
    );

    if (higher_priority_task_woken) {
        portYIELD_FROM_ISR();
    }
}

// Turn LED off after the flash duration
static void led_off_callback(void* arg)
{
    gpio_set_level(LED_PIN, 0);
}

// Flash LED for FLASH_DURATION_MS
static void flash_led(void* arg)
{
    gpio_set_level(LED_PIN, 1);

    // Schedule LED turn-off after 100 ms
    esp_timer_start_once(
        led_off_timer,
        FLASH_DURATION_MS * 1000
    );
}

// Task to handle button presses from the queue
static void button_task(void* arg)
{
    uint32_t io_num;

    for (;;) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {

            // Schedule LED flash 1 second after button press
            esp_timer_stop(flash_timer);
            esp_timer_start_once(
                flash_timer,
                FLASH_DELAY_MS * 1000
            );
        }
    }
}

void app_main(void)
{
    // Configure LED pin as output
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_PIN, 0);

    // Configure button pin as input with pull-up and interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_PIN);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Create queue for passing button events
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    // Create one-shot timer for delayed LED flash
    const esp_timer_create_args_t flash_timer_args = {
        .callback = &flash_led,
        .arg = NULL,
        .name = "flash_timer"
    };
    esp_timer_create(
        &flash_timer_args,
        &flash_timer
    );

    // Create one-shot timer to turn LED off
    const esp_timer_create_args_t led_off_timer_args = {
        .callback = &led_off_callback,
        .arg = NULL,
        .name = "led_off_timer"
    };
    esp_timer_create(
        &led_off_timer_args,
        &led_off_timer
    );

    // Start button handler task
    xTaskCreate(
        button_task,
        "button_task",
        2048,
        NULL,
        10,
        NULL
    );

    // Install ISR service and add handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(
        BUTTON_PIN,
        gpio_isr_handler,
        (void*)BUTTON_PIN
    );
}
```

Now, you can test your project. Good luck!

## Improving the code

ESP-IDF provides a function-like macro `ESP_ERROR_CHECK()`. Here is how it works:

we wrap a function inside `ESP_ERROR_CHECK`, like this:

```
ESP_ERROR_CHECK(esp_timer_create())
```

If `esp_timer_create()` fails, it returns a non-zero return code (this behaviour is coded to the `esp_timer_create()` function itself, as well as many other ESP-IDF functions). If that happens, `ESP_ERROR_CHECK` function returns an error, ESP-IDF logs the error and aborts the program.

If we do not use it, and an error occurs, the firmware continues running. We will never know about the error, and it would be really painful to troubleshoot later.

Replace your `app_main` function with this:

```c
void app_main(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_PIN, 0);

    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_PIN);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    const esp_timer_create_args_t flash_timer_args = {
        .callback = &flash_led,
        .arg = NULL,
        .name = "flash_timer"
    };
    ESP_ERROR_CHECK(esp_timer_create(
        &flash_timer_args,
        &flash_timer
    ));

    const esp_timer_create_args_t led_off_timer_args = {
        .callback = &led_off_callback,
        .arg = NULL,
        .name = "led_off_timer"
    };
    ESP_ERROR_CHECK(esp_timer_create(
        &led_off_timer_args,
        &led_off_timer
    ));

    xTaskCreate(
        button_task,
        "button_task",
        2048,
        NULL,
        10,
        NULL
    );

    ESP_ERROR_CHECK(gpio_install_isr_service(0));
    ESP_ERROR_CHECK(gpio_isr_handler_add(
        BUTTON_PIN,
        gpio_isr_handler,
        (void*)BUTTON_PIN
    ));
}
```

Test your project again to ensure it's working. If the flashes are properly delayed, congratulations!
