# Controlling LED Brightness with PWM

!!! warning "Draft"
    This chapter is not completed yet.

So far, we have only turned our LED fully ON (HIGH voltage) or fully OFF (LOW voltage). But what if you want to set the LED to half brightness, or make it smoothly pulse like a "breathing" light?

Digital pins on the ESP32 can only output either 3.3 V or 0 V. They cannot natively output an intermediate voltage like 1.65 V. To solve this, we use a technique called **Pulse Width Modulation** (or PWM).

## Key concepts

### What is PWM?

It works by switching the GPIO pin between HIGH and LOW thousands of times per second, far faster than human eyes can notice.

Because the switching happens so quickly, your eyes don't see flickering. Instead, they perceive an average brightness depending on how long the pin remains HIGH during each cycle.

#### Key terminologies

- **Duty cycle**: The percentage of time the signal stays HIGH during one cycle. For example:
    - **0% duty cycle**: The pin is always LOW (LED is OFF).
    - **50% duty cycle**: The pin is HIGH for half the time and LOW for half the time (LED appears at half brightness).
    - **100% duty cycle**: The pin is always HIGH (LED is at full brightness).
- **Frequency**: How many duty cycles occur per second (measured in Hertz, Hz). For LEDs, a frequency around 5000 Hz (5 kHz) ensures smooth lighting with zero visible flicker.
- **Duty resolution and duty**: Used for setting the duty cycle. For example:
    - Setting the duty resolution to 10 bits allows $2^{10} = 1024$ duty values (from $0$ to $1023$).
    - Setting the duty to 50% of 1023 (approximately 512) makes the brightness 50% (ON from 0 to 511, OFF from 512 to 1023).

### Other applications of PWM

PWM is not just for dimming LEDs. It is widely used in embedded systems for:

- **Motor speed control**: Adjusting the speed of DC motors in fans or electric vehicles.
- **Servo motors**: Controlling the precise angle of robotic arms.
- **Buzzer audio**: Generating different sound frequencies and tones.

### The ESP32 LEDC peripheral

The ESP32 includes a dedicated hardware module called **LEDC** (LED Control). Once you configure it, the LEDC automatically generates the PWM signal in hardware, requiring **0% CPU usage** to keep running.

The LEDC peripheral uses two main building blocks:

1. **Timer**:
   - Sets the **frequency**.
   - Sets the **bit resolution** or **duty resolution**.
2. **Channel**:
   - Sets the **duty cycle**.
   - Maps the timer signal to a specific physical GPIO pin.

The original ESP32 had 8 timers and 16 channels. 4 timers are low speed mode and the other 4 are high speed mode (each numbered from 0 to 4).

Newer ESP32 only include low-speed mode timers and channels. So, we always use that.

There is another attribute called `hpoint` (high point), which is set to 0 by default, meaning the PWM output turns HIGH right at the beginning of the timer cycle (`count = 0`) and stays HIGH until `count == duty`.

## Hardware required

Everything required for [Blinking an LED](blink.md):

- LED
- Jumper wires
- A 220-330 Ω resistor
- Breadboard

## The code

Let's write a simple program to configure the LEDC peripheral and set our LED to roughly 50% brightness:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "esp_log.h"

void app_main(void)
{
    // 1. Configure the LEDC Timer
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_LOW_SPEED_MODE,
        .timer_num        = LEDC_TIMER_0,
        .duty_resolution  = LEDC_TIMER_10_BIT,
        .freq_hz          = 5000,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ledc_timer_config(&ledc_timer);

    // 2. Configure the LEDC Channel
    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_LOW_SPEED_MODE,
        .channel        = LEDC_CHANNEL_0,
        .timer_sel      = LEDC_TIMER_0,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = GPIO_NUM_22,
        .duty           = 0, // Start with LED turned OFF
        .hpoint         = 0
    };
    ledc_channel_config(&ledc_channel);

    ESP_LOGI("MAIN", "Setting LED to 50% brightness...");

    // Set duty cycle to ~50% (512 out of 1023)
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 512);

    // Update duty cycle to apply changes in hardware
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
}
```

## How it works

1. **Timer setup**: We configure `LEDC_TIMER_0` with a 10-bit resolution (`0` to `1023`) running at 5000 Hz.
2. **Channel setup**: We attach `LEDC_CHANNEL_0` to `GPIO_NUM_22` and tell it to use `LEDC_TIMER_0`.
3. **`ledc_set_duty()`**: Prepares the new brightness value in memory (`512` is half of `1023`).
4. **`ledc_update_duty()`**: Latches the prepared value into hardware logic so the physical pin starts outputting the new PWM signal.

## Improving the code

The standard practice is to define macros for storing configurations like this on top of the main file. By doing this, you will have all configurations in one place:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"
#include "esp_log.h"

#define LEDC_GPIO       GPIO_NUM_22
#define LEDC_MODE       LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL    LEDC_CHANNEL_0
#define LEDC_TIMER      LEDC_TIMER_0
#define LEDC_DUTY_RES   LEDC_TIMER_10_BIT // Resolution: 0 to 1023
#define LEDC_FREQUENCY  5000               // Frequency in Hz (5 kHz)

static const char *TAG = "MAIN";

void app_main(void)
{
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_MODE,
        .timer_num        = LEDC_TIMER,
        .duty_resolution  = LEDC_DUTY_RES,
        .freq_hz          = LEDC_FREQUENCY,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_MODE,
        .channel        = LEDC_CHANNEL,
        .timer_sel      = LEDC_TIMER,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = LEDC_GPIO,
        .duty           = 0,
        .hpoint         = 0
    };
    ledc_channel_config(&ledc_channel);
    
    ESP_LOGI(TAG, "Setting LED to 50%% brightness...");
    ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 512);
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
}
```

## Improving the code further (Breathing light effect)

Setting a static brightness is a good start, but we can continuously adjust the duty cycle in a loop to create a smooth "breathing" light effect:

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/ledc.h"

#define LEDC_GPIO       GPIO_NUM_22
#define LEDC_MODE       LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL    LEDC_CHANNEL_0
#define LEDC_TIMER      LEDC_TIMER_0
#define LEDC_DUTY_RES   LEDC_TIMER_10_BIT // Duty values range from 0 to 1023
#define LEDC_FREQUENCY  5000               // 5 kHz

void app_main(void)
{
    // Configure Timer
    ledc_timer_config_t ledc_timer = {
        .speed_mode       = LEDC_MODE,
        .timer_num        = LEDC_TIMER,
        .duty_resolution  = LEDC_DUTY_RES,
        .freq_hz          = LEDC_FREQUENCY,
        .clk_cfg          = LEDC_AUTO_CLK
    };
    ledc_timer_config(&ledc_timer);

    // Configure Channel
    ledc_channel_config_t ledc_channel = {
        .speed_mode     = LEDC_MODE,
        .channel        = LEDC_CHANNEL,
        .timer_sel      = LEDC_TIMER,
        .intr_type      = LEDC_INTR_DISABLE,
        .gpio_num       = LEDC_GPIO,
        .duty           = 0,
        .hpoint         = 0
    };
    ledc_channel_config(&ledc_channel);

    while (1) {
        // Fade in: gradually increase duty cycle from 0 to 1023
        for (int duty = 0; duty <= 1023; duty += 15) {
            ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty);
            ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
            vTaskDelay(pdMS_TO_TICKS(15));
        }

        // Fade out: gradually decrease duty cycle from 1023 down to 0
        for (int duty = 1023; duty >= 0; duty -= 15) {
            ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, duty);
            ledc_update_duty(LEDC_MODE, LEDC_CHANNEL);
            vTaskDelay(pdMS_TO_TICKS(15));
        }
    }
}
```

Build and flash this code to see your LED smoothly pulsing!

## Test your knowledge

- What is a duty cycle, and how does changing it alter the perceived brightness of an LED?
- What are the roles of the Timer and Channel inside the ESP32's LEDC peripheral?
- If the timer resolution is configured to 8-bit (`LEDC_TIMER_8_BIT`), what duty cycle value corresponds to 100% brightness?
