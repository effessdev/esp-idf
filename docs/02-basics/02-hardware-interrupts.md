# Hardware interrupts

???+ warning "Draft"
    This page is still in the draft stage.

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
    // code...
}
```

`IRAM_ATTR` is a preprocessor macro that makes this function load directly into internal instruction RAM (IRAM), instead of running them from the external flash memory.

**But aren't functions already stored in the RAM?**

On most standard computers, all programs and functions are fully loaded in to the RAM before they run, **but not in ESP32**.

ESP32 (and other MCUs) have very limited RAM (less than 1 MB). So, functions remain in the external Flash memory chip and are read on the fly through a small cache.

## Code

Read the following code from top to bottom to understand it (keep an eye on the comments):

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"

#define LED_PIN       GPIO_NUM_22
#define BUTTON_PIN    GPIO_NUM_23

static int led_state = 0;

// ISR handler: executed immediately on button press (not yet, we have to wire it to the hardware event from the main function)
static void IRAM_ATTR gpio_isr_handler(void* arg)
{
    led_state = !led_state;
    gpio_set_level(LED_PIN, led_state);
}

void app_main(void)
{
    gpio_reset_pin(LED_PIN);
    gpio_set_direction(LED_PIN, GPIO_MODE_OUTPUT);

    // Configure Button as input with pull-up resistor and falling edge interrupt
    gpio_reset_pin(BUTTON_PIN);
    gpio_set_direction(BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_PIN);
    gpio_set_intr_type(BUTTON_PIN, GPIO_INTR_NEGEDGE);

    // Install interrupt service and attach the handler
    gpio_install_isr_service(0);
    gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, NULL);
}
```
