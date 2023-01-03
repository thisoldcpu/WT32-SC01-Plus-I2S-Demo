# WT32-SC01-Plus-I2S-Demo
A simple sketch demonstrating I2S Audio with touch enabled volume control for the WT32-SC01 Plus.

Compiled with Arduino IDE 1.8.19 and tested on a Panlee WT32-SC01 Plus (ZX3D50CE02S-V13) 

Requires:
- AudioI2S
- LovyanGFX
- lvgl

[WIKI](https://github.com/schreibfaul1/ESP32-audioI2S/wiki)

```` c++
#define LGFX_USE_V1

#include <LovyanGFX.hpp>
#include "Arduino.h"
#include "WiFi.h"
#include "Audio.h"

// I2S Controller
#define I2S_LRCK 35
#define I2S_BCLK 36
#define I2S_DOUT 37

// Create Audio objects
Audio audio;

String ssid = "********";
String password = "********";

class LGFX : public lgfx::LGFX_Device
{

    lgfx::Panel_ST7796 _panel_instance;
    lgfx::Bus_Parallel8 _bus_instance; // 8-bit parallel bus instance (ESP32 only)
    lgfx::Light_PWM _light_instance;
    lgfx::Touch_FT5x06 _touch_instance; // FT5206, FT5306, FT5406, FT6206, FT6236, FT6336, FT6436

  public:
    LGFX(void)
    {
      // Bus settings
      {
        auto cfg = _bus_instance.config();
        // 8-bit parallel bus settings
        // cfg.i2s_port = I2S_NUM_0;    // Select the I2S port to use (I2S_NUM_0 or I2S_NUM_1) (use ESP32's I2S LCD mode)
        cfg.freq_write = 20000000;              // Transmit clock (up to 20MHz, rounded to the value obtained by dividing 80MHz by an integer)
        cfg.pin_wr = 47;                        // WR
        cfg.pin_rd = -1;                        // RD
        cfg.pin_rs = 0;                         // RS(D/C)
        cfg.pin_d0 = 9;                         // D0
        cfg.pin_d1 = 46;                        // D1
        cfg.pin_d2 = 3;                         // D2
        cfg.pin_d3 = 8;                         // D3
        cfg.pin_d4 = 18;                        // D4
        cfg.pin_d5 = 17;                        // D5
        cfg.pin_d6 = 16;                        // D6
        cfg.pin_d7 = 15;                        // D7
        _bus_instance.config(cfg);              // Applies the set value to the bus
        _panel_instance.setBus(&_bus_instance); // Set the bus on the panel
      }

      // Display panel settings
      {
        auto cfg = _panel_instance.config();    // Get structure for display panel settings.

        cfg.pin_cs = -1;                        // CS
        cfg.pin_rst = 4;                        // RST
        cfg.pin_busy = -1;                      // BUSY

        // The following setting values â€‹â€‹are general initial values â€‹â€‹for each panel
        cfg.panel_width = 320;    // Actual displayable width
        cfg.panel_height = 480;   // Actual visible height
        cfg.offset_x = 0;         // Panel offset amount in X direction
        cfg.offset_y = 0;         // Panel offset amount in Y direction
        cfg.offset_rotation = 90; // Rotation direction value offset 0~7 (4~7 is upside down)
        cfg.dummy_read_pixel = 8; // Number of bits for dummy read before pixel readout
        cfg.dummy_read_bits = 1;  // Number of bits for dummy read before non-pixel data read
        cfg.readable = true;      // Set to true if data can be read
        cfg.invert = true;        // Set to true if the light/darkness of the panel is reversed
        cfg.rgb_order = false;    // Set to true if the panel's red and blue are swapped
        cfg.dlen_16bit = false;   // Set to true for panels that transmit data length in 16-bit units
        cfg.bus_shared = true;    // If the bus is shared with the SD card, set to True

        // Set the following only when the display is shifted with a driver with a variable number
        // of pixels, such as the ST7735 or ILI9163.
        //    cfg.memory_width     =   240;  // Maximum width supported by the driver IC
        //    cfg.memory_height    =   320;  // Maximum height supported by the driver IC

        _panel_instance.config(cfg);
      }

      // Backlight settings
      {
        auto cfg = _light_instance.config(); // Get structure for backlight settings

        cfg.pin_bl = 45;     // Pin number to which the backlight is connected
        cfg.invert = false;  // True to invert the brightness of the backlight
        cfg.freq = 44100;    // PWM frequency of backlight
        cfg.pwm_channel = 7; // PWM channel number to use

        _light_instance.config(cfg);
        _panel_instance.setLight(&_light_instance); // Set the backlight on display
      }

      // Touch control settings
      {
        auto cfg = _touch_instance.config();

        cfg.x_min = 0;           // Minimum X value (raw value)
        cfg.x_max = 319;         // Maximum X value (raw value)
        cfg.y_min = 0;           // Minimum Y value (raw value)
        cfg.y_max = 479;         // Maximum Y value (raw value)
        cfg.pin_int = 7;         // INT
        cfg.bus_shared = true;   // Set to True when using a common bus with the screen
        cfg.offset_rotation = 0; // Adjustment when the display and touch direction do not match (Set with a value of 0 to 7)
        // For I2C connection
        cfg.i2c_port = 1;        // Sel ect I2C to use (0 or 1)
        cfg.i2c_addr = 0x38;     // I2C device address number
        cfg.pin_sda = 6;         // SDA
        cfg.pin_scl = 5;         // SCL
        cfg.freq = 400000;       // Set I2C clock

        _touch_instance.config(cfg);
        _panel_instance.setTouch(&_touch_instance); // Set the touch screen on the panel
      }

      setPanel(&_panel_instance); // Set the panel to use
    }
};

// Create an instance of the prepared class
#include <lvgl.h>

LGFX tft;

#define screenWidth 480
#define screenHeight 320

static lv_disp_draw_buf_t draw_buf;
static lv_color_t buf[screenWidth * 10];

void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p)
{
  uint32_t w = (area->x2 - area->x1 + 1);
  uint32_t h = (area->y2 - area->y1 + 1);
  tft.startWrite();
  tft.setAddrWindow(area->x1, area->y1, w, h);
  tft.writePixels((lgfx::rgb565_t *)&color_p->full, w * h);
  tft.endWrite();
  lv_disp_flush_ready(disp);
}

void my_touch_read(lv_indev_drv_t *indev_driver, lv_indev_data_t *data)
{
  uint16_t touchX, touchY;
  bool touched = tft.getTouch(&touchX, &touchY);
  if (!touched)
  {
    data->state = LV_INDEV_STATE_REL;
  }
  else
  {
    data->state = LV_INDEV_STATE_PR;
    data->point.x = touchX;
    data->point.y = touchY;
  }
}

static lv_style_t label_style;
static lv_obj_t *label;

static void slider_event_cb(lv_event_t *e)
{
  lv_obj_t *slider = lv_event_get_target(e);

  // Refresh the text
  lv_label_set_text_fmt(label, "%" LV_PRId32, lv_slider_get_value(slider));
  lv_obj_align_to(label, slider, LV_ALIGN_OUT_TOP_MID, 0, -15); /*Align top of the slider*/

  audio.setVolume(lv_slider_get_value(slider) / 5);
}

// Create a slider and write its value on a label.
void CreateControls(void)
{
  // Create a slider in the center of the display
  lv_obj_t *slider = lv_slider_create(lv_scr_act());
  lv_obj_set_width(slider, 210);                                              // Set the width
  lv_obj_center(slider);                                                      // Align to the center of the parent (screen)
  lv_obj_add_event_cb(slider, slider_event_cb, LV_EVENT_VALUE_CHANGED, NULL); // Assign an event function

  // Create a label above the slider
  label = lv_label_create(lv_scr_act());
  lv_label_set_text(label, "0");
  lv_obj_align_to(label, slider, LV_ALIGN_OUT_TOP_MID, 0, -15); // Align top of the slider
}

void setup() {
  tft.begin();
  tft.setRotation(3);
  tft.setBrightness(255);

  lv_init();
  lv_disp_draw_buf_init(&draw_buf, buf, NULL, screenWidth * 10);
  static lv_disp_drv_t disp_drv;
  lv_disp_drv_init(&disp_drv);
  disp_drv.hor_res = screenWidth;
  disp_drv.ver_res = screenHeight;
  disp_drv.flush_cb = my_disp_flush;
  disp_drv.draw_buf = &draw_buf;
  lv_disp_drv_register(&disp_drv);
  static lv_indev_drv_t indev_drv;
  lv_indev_drv_init(&indev_drv);
  indev_drv.type = LV_INDEV_TYPE_POINTER;
  indev_drv.read_cb = my_touch_read;
  lv_indev_drv_register(&indev_drv);

  CreateControls();

  WiFi.disconnect();
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), password.c_str());
  while (WiFi.status() != WL_CONNECTED) delay(1500);

  audio.setPinout(I2S_BCLK, I2S_LRCK, I2S_DOUT);
  audio.setVolume(10);                                                   // Values from 0 to 21

  audio.connecttohost("http://us5.internet-radio.com:8201/listen.pls");
}

void loop() {
  audio.loop();

  lv_timer_handler();
}

````
