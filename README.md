- 👋 Hi, I’m @Talla1709
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
Talla1709/Talla1709 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
I am working on the esp32 wrover microcontroller and I have to light a led on this microcontroller with two capacitive sensors, I have followed all the steps and it does not work
.I send you below the code I wrote but it does not work, can someone help me? 


/*the main steps of this work

Initialization of touch pad driver

Configuration of touch pad GPIO pins

Taking measurements

Adjusting parameters of measurements

Filtering measurements

Touch detection methods

Setting up interrupts to report touch detection

Waking up from Sleep mode on interrupt*/



#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "driver/touch_pad.h"
#include "esp_log.h"
#include "soc/rtc_periph.h"
#include "soc/sens_periph.h"
#include "sdkconfig.h"
#include "driver/gpio.h"



/*#define GPIO_OUTPUT_IO_0 2 // defined as output IO Number
#define GPIO_OUTPUT_IO_1
#define GPIO_OUTPUT_PIN_SEL  ((1ULL<<GPIO_OUTPUT_IO_0) | (1ULL<<GPIO_OUTPUT_IO_1)) //// Open pin flag ,1UL To open ,0UL To close
#define GPIO_INPUT_T0 4 // Connected to GPIO 4
#define GPIO_INPUT_T5 14 // connected to GPIO 14
#define GPIO_INPUT_PIN_SEL  ((1ULL<<GPIO_INPUT_T0) | (1ULL<<GPIO_INPUT_T5))
*/
#define TOUCH_BUTTON_NUM 2
#define TOUCH_CHANGE_CONFIG 0
// #define TOUCH_PAD_MAX 10
#define TOUCH_THRESH_NO_USE   (0)
#define TOUCH_PAD_NO_CHANGE   (-1)
#define TOUCH_FILTER_MODE_EN  (1)
#define TOUCHPAD_FILTER_TOUCH_PERIOD (10)
#define TOUCH_THRESH_PERCENT  (80)
#define LED_PIN 2

static const char *TAG = "touch read";

static bool s_pad_activated[TOUCH_BUTTON_NUM];
static uint32_t s_pad_init_val[TOUCH_BUTTON_NUM];
static xQueueHandle touchQueueStatic;
/*
  Read values sensed at all available touch pads.
  Use 2 / 3 of read value as the threshold
  to trigger interrupt when the pad is touched.
  Note: this routine demonstrates a simple way
  to configure activation threshold for the touch pads.
  Do not touch any pads when this routine
  is running (on application start).
 */




static void tp_example_set_thresholds(void)

{
    uint16_t touch_value;
    for (int i = 0; i < TOUCH_BUTTON_NUM; i++)
    {
        //read filtered value
        touch_pad_read_filtered(i, &touch_value);
        s_pad_init_val[i] = touch_value;
        ESP_LOGI(TAG, "test init: touch pad [%d] val is %d", i, touch_value);
        //set interrupt threshold.
        ESP_ERROR_CHECK(touch_pad_set_thresh(i, touch_value * 2 / 3));

    }
}

/*
  Check if any of touch pads has been activated
  by reading a table updated by rtc_intr()
  If so, then print it out on a serial monitor.
  Clear related entry in the table afterwards
  In interrupt mode, the table is updated in touch ISR.
  In filter mode, we will compare the current filtered value with the initial one.
  If the current filtered value is less than 80% of the initial value, we can
  regard it as a 'touched' event.
  When calling touch_pad_init, a timer will be started to run the filter.
  This mode is designed for the situation that the pad is covered
  by a 2-or-3-mm-thick medium, usually glass or plastic.
  The difference caused by a 'touch' action could be very small, but we can still use
  filter mode to detect a 'touch' event.
 */


static void tp_example_read_task(void *pvParameter)
{
    static int show_message;
    int change_mode = 0;
    int filter_mode = 1;
    //touch_trigger_mode_t triggerMode;

    while (1) {
        if (filter_mode == 0) {
            //interrupt mode, enable touch interrupt
            touch_pad_intr_enable();
            for (int i = 0; i < TOUCH_BUTTON_NUM; i++) {
                if (s_pad_activated[i] == true) {
                    ESP_LOGI(TAG, "T%d activated!", i);
                    // Wait a while for the pad being released
                    vTaskDelay(200 / portTICK_PERIOD_MS);
                    // Clear information on pad activation
                    s_pad_activated[i] = false;
                    // Reset the counter triggering a message
                    // that application is running
                    show_message = 1;
                }
            }
        } else {
            //filter mode, disable touch interrupt
            touch_pad_intr_disable();
            touch_pad_clear_status();
            for (int i = 0; i < TOUCH_BUTTON_NUM; i++) {
                uint16_t value = 0;
                touch_pad_read_filtered(i, &value);
                if (value < s_pad_init_val[i] * TOUCH_THRESH_PERCENT / 100) {
                    ESP_LOGI(TAG, "T%d activated!", i);
                    ESP_LOGI(TAG, "value: %d; init val: %d", value, s_pad_init_val[i]);
                    vTaskDelay(200 / portTICK_PERIOD_MS);
                    // Reset the counter to stop changing mode.
                    change_mode = 1;
                    show_message = 1;
                }
            }
        }

        vTaskDelay(10 / portTICK_PERIOD_MS);

        // If no pad is touched, every couple of seconds, show a message
        // that application is running
        if (show_message++ % 500 == 0) {
            ESP_LOGI(TAG, "Waiting for any pad being touched...");
            //touch_pad_get_trigger_mode(&triggerMode);
        }
//        if (triggerMode==TOUCH_TRIGGER_BELOW)
//        {
//                    ESP_LOGI(TAG, "Switching trigger mode to ABOVE");
//
//                    touch_pad_set_trigger_mode(TOUCH_TRIGGER_ABOVE);
//           }
//        else {
//                	ESP_LOGI(TAG, "Switching trigger mode to BELOW");
//
//                    touch_pad_set_trigger_mode(TOUCH_TRIGGER_BELOW);
//            }
        // Change mode if no pad is touched for a long time.
        // We can compare the two different mode.
        if (change_mode++ % 2000 == 0) {
            filter_mode = !filter_mode;
            ESP_LOGW(TAG, "Change mode...%s", filter_mode == 0 ? "interrupt mode" : "filter mode");
        }
    }

/*Handle an interrupt triggered when a pad is touched.
  Recognize what pad has been touched and save it in a table.
 */
static void tp_example_rtc_intr (void *arg)


   //static void tp_example_rtc_intr(void)
{
    	BaseType_t xHigherPriorityTaskWoken;
    	 /* We have not woken a task at the start of the ISR. */
    	 xHigherPriorityTaskWoken = pdFALSE;
    uint32_t pad_intr = touch_pad_get_status();
    //clear interrupt
    touch_pad_clear_status();
    for (int i = 0; i < TOUCH_BUTTON_NUM; i++) {
        if ((pad_intr >> i) & 0x01) {
        	 xQueueSendFromISR(touchQueueStatic, (void *)i, NULL);
        	s_pad_activated[i] = true;
        }
    }
}


// Before reading touch pad, we need to initialize the RTC IO.

    static void tp_example_touch_pad_init (void)
    {
    for (int i = 0; i < TOUCH_BUTTON_NUM; i++)
    {
        //init RTC IO and mode for touch pad.
        touch_pad_config(i, TOUCH_THRESH_NO_USE);
    }
}


    void app_main()
{


    // / Initialize touch pad peripheral, it will start a timer to run a filter
        ESP_LOGI(TAG, "Initializing touch pad");
        ESP_ERROR_CHECK(touch_pad_init());

        // If use interrupt trigger mode, should set touch sensor FSM mode at 'TOUCH_FSM_MODE_TIMER'
                touch_pad_set_fsm_mode(TOUCH_FSM_MODE_TIMER);
                touch_pad_set_voltage(TOUCH_HVOLT_2V4, TOUCH_LVOLT_0V8, TOUCH_HVOLT_ATTEN_1V5);
                tp_example_touch_pad_init(); // Init touch pad IO
                // Initialize and start a software filter to detect slight change of capacitance.
                touch_pad_filter_start(100);
                // Set thresh hold
                    tp_example_set_thresholds();

                touch_pad_set_trigger_mode(TOUCH_TRIGGER_BELOW);


        //Set measuring time and sleep time

           // In this case, measurement will sustain 0xffff / 8Mhz = 8.19ms
           // Meanwhile, sleep time between two measurement will be 0x1000 / 150Khz = 27.3 ms
    /* If you want change the touch sensor default setting, please write here(after initialize). There are examples: */
        touch_pad_set_meas_time(0x1000, 0xffff);

        /*touch_pad_set_idle_channel_connect(TOUCH_PAD_IDLE_CH_CONNECT_DEFAULT);


        for (int i = 0; i < TOUCH_BUTTON_NUM; i++)
        {
            touch_pad_set_cnt_mode(button[i], TOUCH_PAD_SLOPE_DEFAULT, TOUCH_PAD_TIE_OPT_DEFAULT);
        }

        // Denoise setting at TouchSensor 0.

        touch_pad_denoise_t denoise = {
          // The bits to be cancelled are determined according to the noise level.
          .grade = TOUCH_PAD_DENOISE_BIT4,
          .cap_level = TOUCH_PAD_DENOISE_CAP_L4,
        };
        touch_pad_denoise_set_config(&denoise);
        touch_pad_denoise_enable();
        ESP_LOGI(TAG, "Denoise function init");
*/

        //touch_pad_fsm_start();



    //
    // Register touch interrupt ISR
        touch_pad_isr_register(tp_example_rtc_intr, NULL);

    // Start a task to show what pads have been touched
    xTaskCreate(&tp_example_read_task,"touch_pad_read_task",2048, NULL,5,NULL);
    touchQueueStatic = xQueueCreate(10, sizeof(int));
}
}

