# Hardware interrupts

Hardware interrupts are hardware signals that instantly pause the normal program flow to handle external events such as a button press, without wasting CPU cycles on continuous polling.

Let's update our code such that the LED can be toggled using a button press.

The connections are the same as before (LED & resistor). But we have to connect a jumper wire to GPIO23. Leave the other end of the jumper wire not connected to anything.

## Key concepts

### ISR handler

ISR handler stands for **Interrupt Service Routine (ISR) handler**. It's a function that we configure to be run immediately when a hardware event occurs (in this case, the button press).

### `IRAM_ATTR`

While creating ISRs, we prefix `IRAM_ATTR` before the function name:

```c
static void IRAM_ATTR my_isr_handler(void* arg)
{
    // code to toggle the LED
}
```

`IRAM_ATTR` is a preprocessor macro that makes this function load directly into internal instruction RAM (IRAM), instead of running them from the external flash memory.

**But aren't functions already stored in the RAM?**

On most standard computers, all programs and functions are fully loaded in to the RAM before they run, **but not in ESP32**.

ESP32 (and other MCUs) have very limited RAM (less than 1 MB). So, functions remain in the external Flash memory chip and are read on the fly through a small cache.

## How we are going to do this (conceptually)

- We set the button GPIO pin to input mode, which will allow the ESP32 to read the voltage in the button pin.
- We give a constant HIGH voltage to the button pin.
- We put a resistor (also called pull-up resistor) between the voltage source and the pin, allowing us to connect the pin to GND (zero voltage) without causing high current.
- We configure the button pin to trigger a signal (interrupt) when it goes from HIGH to LOW.
- Now, when we touch the other end of the wire to GND, the voltage of the button pin will drop to LOW, triggering the interrupt, and causing the ISR handler to execute.

## The actual code

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define LED_PIN       GPIO_NUM_22
#define BUTTON_PIN    GPIO_NUM_23

static int led_state = 0;

// Define the ISR handler
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    led_state = !led_state;
    gpio_set_level(LED_PIN, led_state);
}

void app_main(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    gpio_reset_pin(BUTTON_PIN);

    // Set button pin to input mode
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    
    // Enable pull-up resistor in button pin
    // (this will also give a HIGH voltage to the pin)
    gpio_pullup_en(BUTTON_PIN);

    // Set the interrupt type to GPIO_INTR_NEGEDGE, which
    // stands for "when the voltage go from HIGH to LOW"
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Install interrupt service (required for properly
    // handing the interrupt over to the ISR handler)
    gpio_install_isr_service(0);

    // Attach the ISR handler to the button pin
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, NULL);
}
```

## Test the code

Now, after building and flashing, touch the button pin to GND. If you did everything correctly, the LED will respond to the input.

## Improving the code

Currently, we are toggling the LED directly inside the ISR handler. This is not recommended because ISRs have the highest priority, and will block anything else that's running until it finishes.

Instead, we use a pattern where we:

- Create a queue
- Create a function which runs in parallel to the main function when started from the main function (we call it a task), which continuously consume the items in the queue
- Make the ISR handler add an event item to the queue, which the button task will handle one by one (first added is first handled)

Here is the code that uses this approach:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/gpio.h"

#define LED_PIN       GPIO_NUM_22
#define BUTTON_PIN    GPIO_NUM_23

// Queue to send button events from ISR to the task
static QueueHandle_t gpio_evt_queue = NULL;

// ISR keeps work minimal (add the event to the queue)
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    uint32_t gpio_num = (uint32_t) arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
}

// Task to handle button presses from the queue
static void button_task(void* arg)
{
    uint32_t io_num;
    int led_state = 0;

    for (;;) {
        // xQueueRecieve recieves the first item added to the queue.
        // If there are no items, it waits until one is added.
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            // Toggle LED state on button press event
            led_state = !led_state;
            gpio_set_level(LED_PIN, led_state);
            
            // Simple software debounce delay
            vTaskDelay(pdMS_TO_TICKS(200));
        }
    }
}

void app_main(void)
{
    // Configure LED Pin as Output
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button Pin as Input with Pull-Up and Interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_PIN);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Create Queue for Passing Events
    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));

    // Start Button Handler Task
    xTaskCreate(button_task, "button_task", 2048, NULL, 10, NULL);

    // Install ISR Service and Add Handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, (void*) BUTTON_PIN);
}
```

Now, our code is much better!
