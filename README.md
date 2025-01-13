# nrf_custom_gatt_notification

------
## 선언
```
#define CUSTOM_SERVICE_UUID        0x1400  // Custom Service UUID (bonding evt)
#define CUSTOM_CHAR_UUID           0x1401  // Custom Characteristic UUID

typedef struct
{
    uint16_t                     service_handle;       // Handle for the GATT service
    ble_gatts_char_handles_t     char_handles;         // Handles for the custom characteristic
    uint16_t                     conn_handle;          // Connection handle
} custom_service_t;

static custom_service_t m_custom_service;

```

------
## init

init은 ble_stack_init 함수보다 아래에서 실행되어야 함(soft device 초기화)  

```
void custom_service_init(custom_service_t *p_service)
{
    ret_code_t err_code;
    ble_uuid_t service_uuid;

    // Initialize the service structure
    p_service->conn_handle = BLE_CONN_HANDLE_INVALID;

    // Set the service UUID
    service_uuid.uuid = CUSTOM_SERVICE_UUID;
    service_uuid.type = BLE_UUID_TYPE_BLE;

    // Add the service to the BLE stack
    err_code = sd_ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &service_uuid, &p_service->service_handle);
    APP_ERROR_CHECK(err_code);

    // Add the characteristic
    ble_add_char_params_t char_params;
    memset(&char_params, 0, sizeof(char_params)); 
    char_params.uuid              = CUSTOM_CHAR_UUID;
    char_params.uuid_type         = BLE_UUID_TYPE_BLE;
    char_params.max_len           = 20;  // Maximum data length 
    char_params.init_len          = sizeof(uint8_t);
    char_params.char_props.notify = 1;   // Enable Notifications
    char_params.cccd_write_access = SEC_OPEN;

    err_code = characteristic_add(p_service->service_handle, &char_params, &p_service->char_handles);
    APP_ERROR_CHECK(err_code);

    NRF_LOG_INFO("Custom Service initialized."); 
}
```
--------
## notify 전송 
```

ret_code_t custom_notification_send(custom_service_t * p_service, uint8_t * p_data, uint16_t len)
{
    if (p_service->conn_handle == BLE_CONN_HANDLE_INVALID)
    {
        printf("No active connection. Unable to send notification.\r\n");
        return NRF_ERROR_INVALID_STATE;
    }

    // CCCD 활성화 상태 확인
    ble_gatts_value_t gatts_value;
    uint8_t cccd_value[2] = {0};
    memset(&gatts_value, 0, sizeof(gatts_value));

    gatts_value.p_value = cccd_value;
    gatts_value.len = sizeof(cccd_value);

    ret_code_t err_code = sd_ble_gatts_value_get(p_service->conn_handle, p_service->char_handles.cccd_handle, &gatts_value);
    if (err_code != NRF_SUCCESS || cccd_value[0] != BLE_GATT_HVX_NOTIFICATION)
    {
        printf("CCCD not enabled. Notification cannot be sent.\r\n");
        return NRF_ERROR_INVALID_STATE;
    }

    // Notification 전송
    ble_gatts_hvx_params_t hvx_params;
    memset(&hvx_params, 0, sizeof(hvx_params));

    hvx_params.handle = p_service->char_handles.value_handle;
    hvx_params.type   = BLE_GATT_HVX_NOTIFICATION;
    hvx_params.p_len  = &len;
    hvx_params.p_data = p_data;

    err_code = sd_ble_gatts_hvx(p_service->conn_handle, &hvx_params);
    if (err_code != NRF_SUCCESS)
    {
        printf("Notification failed: %d\r\n", err_code);
    }
    else
    {
        printf("Notification sent successfully.\r\n");
    }

    return err_code;
}
```
