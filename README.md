#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "lwip/ip4_addr.h"

#define REQUEST_GPIO 4
#define GRANT_GPIO 5
#define DELAY_US 10
#define REQUEST_PERIOD_MS 100

#define WIFI_SSID "NETGEAR70"
#define WIFI_PASS "boldcanon345"

#define STATIC_IP   "192.168.1.200"
#define GATEWAY_IP  "192.168.1.1"
#define NETMASK_IP  "255.255.255.0"

static const char *TAG = "GPIO_WIFI";

void wifi_init_sta(void) {
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_t *netif = esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    esp_netif_ip_info_t ip_info;
    ip4_addr_t tmp_ip;

    ip4addr_aton(STATIC_IP, &tmp_ip);
    ip_info.ip.addr = tmp_ip.addr;

    ip4addr_aton(GATEWAY_IP, &tmp_ip);
    ip_info.gw.addr = tmp_ip.addr;

    ip4addr_aton(NETMASK_IP, &tmp_ip);
    ip_info.netmask.addr = tmp_ip.addr;

    ESP_ERROR_CHECK(esp_netif_dhcpc_stop(netif));
    ESP_ERROR_CHECK(esp_netif_set_ip_info(netif, &ip_info));

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
    ESP_LOGI("WiFi", "Static IP: " IPSTR, IP2STR(&ip_info.ip));
}

void request_grant_task(void *pvParameter) {
    while (1) {
        GPIO.out_w1ts.val = (1 << REQUEST_GPIO);
        esp_rom_delay_us(DELAY_US);
        GPIO.out_w1tc.val = (1 << GRANT_GPIO);

        esp_rom_delay_us(DELAY_US);

        GPIO.out_w1tc.val = (1 << REQUEST_GPIO);
        GPIO.out_w1ts.val = (1 << GRANT_GPIO);

        vTaskDelay(pdMS_TO_TICKS(REQUEST_PERIOD_MS));
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing...");

    ESP_ERROR_CHECK(nvs_flash_init());
    wifi_init_sta();

    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
