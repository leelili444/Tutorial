---
description: flow of IMU data to attitude estimation and telemetry transmission
---
## Overview
Purpose: read IMU (ICM42688P) → compute attitude (AHRS) → pack and send telemetry frames  via UART DMA.

Main files
- `Core/Src/ICM42688P.c` — sensor driver, DMA/ISR read, decode to `imudata`.
- `Core/Src/ins_task.c` — calibration, gyroscope offset, AHRS → produces `euler`.
- `Core/Src/mav_task.c` — periodic packing and UART DMA transmit (ping‑pong buffers).
- `Core/Src/main.c` — hardware init, DMA/EXTI callbacks, task creation.

## Key global data
- `ICM42688P_Data_t imudata` — sensor sample (accel / gyro / temp / dt).  
- `FusionEuler euler` — AHRS output (roll / pitch / yaw).  
- `IMU_Frame_t tx_buffers[2]`, `volatile dma_ready`, `active_buf_idx` — transmit buffers and DMA state.

---

## ins_task — structure, responsibilities, and key code

Responsibilities
- Apply sensor calibration.
- Estimate gyroscope offset (FusionOffset).
- Run AHRS (no magnetometer) with correct dt.
- Publish/write `euler` for other tasks.

Key excerpt (from `Core/Src/ins_task.c`):

```
void InitializePose(void){
    FusionOffsetInitialise(&offset, SAMPLE_RATE);
    FusionAhrsInitialise(&ahrs);
    const FusionAhrsSettings settings = {
        .convention = FusionConventionNwu,
        .gain = 0.5f,
        .gyroscopeRange = 2000.0f,
        .accelerationRejection = 10.0f,
        .magneticRejection = 10.0f,
        .recoveryTriggerPeriod = 5 * SAMPLE_RATE,
    };
    FusionAhrsSetSettings(&ahrs, &settings);
}

void GetPose(ICM42688P_Data_t *imudata){
    FusionVector gyroscope = {{imudata->gyro_x, imudata->gyro_y, imudata->gyro_z}};
    FusionVector accelerometer = {{imudata->accel_x, imudata->accel_y, imudata->accel_z}};
    gyroscope = FusionCalibrationInertial(...);
    accelerometer = FusionCalibrationInertial(...);
    gyroscope = FusionOffsetUpdate(&offset, gyroscope);
    if (offset.gyroscopeOffset.axis.z != 0) {
        FusionAhrsUpdateNoMagnetometer(&ahrs, gyroscope, accelerometer, 1e-6f * imudata->dt);
        euler = FusionQuaternionToEuler(FusionAhrsGetQuaternion(&ahrs));
    }
}
```

Notes
- `imudata->dt` is in microseconds; AHRS expects seconds (`1e-6f * dt`).
- `GetPose` must receive a consistent snapshot of `imudata`.

---

## mav_task — structure, responsibilities, and key code

Responsibilities
- Periodic (≈1 ms) snapshot of `imudata` + `euler`, pack telemetry frame, start UART DMA.
- Use ping‑pong buffers and `dma_ready` to avoid collisions.
- Include time delta (`dt`) and CRC in frames.

Key excerpts (from `Core/Src/mav_task.c`):

```
void Telemetry_Send(ICM42688P_Data_t* imudata, FusionEuler* euler) {
    if (dma_ready == 0) return;
    IMU_Frame_t *p_pkt = &tx_buffers[active_buf_idx];
    p_pkt->counter = frame_tick++;
    p_pkt->dt = delta_us;
    p_pkt->ax = imudata->accel_x; p_pkt->ay = imudata->accel_y; p_pkt->az = imudata->accel_z;
    p_pkt->gx = imudata->gyro_x;  p_pkt->gy = imudata->gyro_y;  p_pkt->gz = imudata->gyro_z;
    p_pkt->roll = euler->angle.roll; p_pkt->pitch = euler->angle.pitch; p_pkt->yaw = euler->angle.yaw;
    __HAL_CRC_DR_RESET(&hcrc);
    p_pkt->crc32 = HAL_CRC_Calculate(&hcrc, (uint32_t*)p_pkt, 7);
    dma_ready = 0;
    HAL_StatusTypeDef status = HAL_UART_Transmit_DMA(&huart1, (uint8_t*)p_pkt, sizeof(IMU_Frame_t));
    if (status != HAL_OK) dma_ready = 1;
    else active_buf_idx = !active_buf_idx;
}
```

Periodic task loop snapshot:

```
void mavlinkTask(void *argument) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xFrequency = pdMS_TO_TICKS(2); // ~2ms
    for(;;) {
        vTaskDelayUntil(&xLastWakeTime, xFrequency);
        ICM42688P_Data_t local_imu;
        FusionEuler local_euler;
        taskENTER_CRITICAL();
        local_imu = imudata;
        local_euler = euler;
        taskEXIT_CRITICAL();
        Telemetry_Send(&local_imu, &local_euler);
        // update delta_us from TIM counter...
    }
}
```

Notes
- If DMA busy, frame is skipped (no blocking).
- `HAL_UART_TxCpltCallback` must set `dma_ready = 1` when done.

---

## Callbacks and task interaction (examples from `Core/Src/main.c`)

SPI DMA complete ISR decodes sample and notifies INS task:

```
void HAL_SPI_TxRxCpltCallback(SPI_HandleTypeDef *hspi) {
  if (hspi->Instance == SPI1) {
    Icm_CS_HIGH();
    ICM42688P_decode(&imudata);
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    if (insTaskHandle != NULL) {
        vTaskNotifyGiveFromISR(insTaskHandle, &xHigherPriorityTaskWoken);
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
  }
}
```

UART TX complete should release DMA flag:

```
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        dma_ready = 1;
    }
}
```

---

## Data flow (step by step)
1. SPI DMA read → ISR: `ICM42688P_decode(&imudata)` → `vTaskNotifyGiveFromISR(insTaskHandle)`.  
2. INS task wakes, atomically copies `imudata` → `GetPose(local_imu)` → updates `euler`.  
3. MAV task periodically snapshots `imudata`+`euler` (atomic) → `Telemetry_Send(local copies)` → UART DMA transmit.  
4. UART DMA complete ISR sets `dma_ready = 1`.

---

## Concurrency & synchronization — why atomic snapshots
- `imudata` and `euler` are shared between ISRs and tasks. A struct copy can be interrupted and produce inconsistent fields.
- Current solution: very short critical sections (`taskENTER_CRITICAL()` / `taskEXIT_CRITICAL()`) to copy shared structures into local variables before use.
- Keep critical sections minimal; avoid blocking calls inside them.

Alternatives
- Use FreeRTOS queues: ISR calls `xQueueSendFromISR()` with full sample → task receives.
- Double-buffer with explicit state flags and memory barriers.

---

## Wrap up
Today we covered the attitude estimation and serial transmission modules, including task structure, key code excerpts, and concurrency considerations.

