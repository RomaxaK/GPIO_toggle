#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "esp_system.h"

// Wi-Fi includes
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_netif_ip_addr.h"

// GPIO config
#define REQUEST_GPIO 4
#define GRANT_GPIO 5
#define DELAY_US 1
#define REQUEST_PERIOD_MS 100

// Wi-Fi credentials
#define WIFI_SSID "YourSSID"
#define WIFI_PASS "YourPassword"

// Static IP config (adjust for your network)
#define STATIC_IP   "192.168.4.50"
#define GATEWAY_IP  "192.168.4.1"
#define NETMASK_IP  "255.255.255.0"

static const char *TAG = "GPIO_WIFI";

// -------- Wi-Fi Init with Static IP --------
void wifi_init_sta(void) {
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_t *netif = esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // Set static IP
    esp_netif_ip_info_t ip_info;
    ip4addr_aton(STATIC_IP, &ip_info.ip);
    ip4addr_aton(GATEWAY_IP, &ip_info.gw);
    ip4addr_aton(NETMASK_IP, &ip_info.netmask);
    ESP_ERROR_CHECK(esp_netif_dhcpc_stop(netif));
    ESP_ERROR_CHECK(esp_netif_set_ip_info(netif, &ip_info));

    // Wi-Fi config
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI("WiFi", "Connecting to %s...", WIFI_SSID);

    // Log IP info
    ESP_LOGI("WiFi", "Static IP: " IPSTR, IP2STR(&ip_info.ip));
}

// -------- GPIO Task (Grant is inverse of Request) --------
void request_grant_task(void *pvParameter) {
    int priority = *(int*)pvParameter;

    while (1) {
        // REQUEST HIGH, GRANT LOW
        GPIO.out_w1ts.val = (1 << REQUEST_GPIO);   // Set REQUEST high
        GPIO.out_w1tc.val = (1 << GRANT_GPIO);     // Set GRANT low
        esp_rom_delay_us(DELAY_US);

        // REQUEST LOW, GRANT HIGH
        GPIO.out_w1tc.val = (1 << REQUEST_GPIO);   // Set REQUEST low
        GPIO.out_w1ts.val = (1 << GRANT_GPIO);     // Set GRANT high

        vTaskDelay(pdMS_TO_TICKS(REQUEST_PERIOD_MS));
    }
}

// -------- Main App Entry Point --------
void app_main(void) {
    ESP_LOGI(TAG, "Initializing...");

    ESP_ERROR_CHECK(nvs_flash_init());
    wifi_init_sta();

    // Configure GPIOs
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    // Start with GRANT HIGH
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
