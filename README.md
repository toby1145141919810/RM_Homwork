# 电控组第三次培训作业
- **Chen  ChangYu**
- ***20263001691***

## 一、功能实现说明
本项目基于 STM32F103C8T6 实现了串口指令控制流水灯系统，包含以下核心功能：
1. **串口通信与协议解析**：使用串口接收中断（USART1，波特率 115200），利用状态机解析自定义数据包 `0xAA RUN DIR STEP_L STEP_H CHECK`。包含帧头检测与累加和校验，保证通信的可靠性。
2. **非阻塞流水灯状态机**：基于 `HAL_GetTick()` 实现非阻塞延时，根据指令动态调整运行/停止、正向/反向及切换间隔时间（Step），不占用 CPU，不影响按键和串口的实时响应。
3. **按键非阻塞消抖**：采用时间戳进行按键消抖，物理按键可本地控制运行状态（KEY0）和方向（KEY2）。
4. **OLED 状态显示**：使用软件 I2C（PB0-SDA, PB1-SCL）驱动 OLED 屏幕，实时显示当前 `RUN`、`DIR`、`STEP`、`LED` 状态。
5. **双向串口反馈**：无论是通过上位机下发指令，还是本地按键改变状态，单片机均会通过串口向上位机发送当前状态字符串（如 `OK: RUN=1, DIR=0, STEP=500, LED=1`）。

## 二、自定义通信协议
| 字节位置 | 含义 | 说明 |
| :--- | :--- | :--- |
| Byte 0 | 帧头 | 固定 `0xAA` |
| Byte 1 | 运行状态 (RUN) | `0x00` 停止，`0x01` 运行 |
| Byte 2 | 方向 (DIR) | `0x00` 正向，`0x01` 反向 |
| Byte 3 | 步长低八位 (STEP_L) | 与 STEP_H 组合为 16 位无符号整数 |
| Byte 4 | 步长高八位 (STEP_H) | 单位：毫秒 (ms) |
| Byte 5 | 校验和 (CHECK) | 前 5 个字节累加和的低 8 位 |

## 三、核心代码片段

### 1. 串口接收状态机 (数据包解析与同步反馈)
```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        // 状态机解析数据包，格式：AA RUN DIR STEP_L STEP_H CHECK
        if (rx_cnt == 0 && rx_data != 0xAA) {
            // 没收到帧头，丢弃
        } else {
            rx_buf[rx_cnt++] = rx_data;
            if (rx_cnt == 6) {
                rx_cnt = 0; // 清空计数器
                // 计算校验和
                uint8_t check_sum = rx_buf[0] + rx_buf[1] + rx_buf[2] + rx_buf[3] + rx_buf[4];
                if (check_sum == rx_buf[5]) {
                    run_flag = rx_buf[1];
                    dir_flag = rx_buf[2];
                    step_ms = (rx_buf[4] << 8) | rx_buf[3];
                    
                    // 串口反馈
                    char msg[50];
                    sprintf(msg, "OK: RUN=%d, DIR=%d, STEP=%d\r\n", run_flag, dir_flag, step_ms);
                    HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), 100);
                }
            }
        }
        HAL_UART_Receive_IT(&huart1, &rx_data, 1); // 重新开启接收
    }
}

2. 非阻塞流水灯逻辑


3.按键非阻塞消抖 (以 KEY0 为例)
if (run_flag) {
    if (HAL_GetTick() - last_led_time >= step_ms) {
        last_led_time = HAL_GetTick();
        if (dir_flag == 0) { // 正向
            current_led = (current_led + 1) % 3;
        } else { // 反向
            current_led = (current_led == 0) ? 2 : current_led - 1;
        }
    }
}


if (HAL_GPIO_ReadPin(KEY0_GPIO_Port, KEY0_Pin) == GPIO_PIN_RESET && !key0_lock) {
    if (HAL_GetTick() - key0_time > 20) {
        key0_lock = 1;
        run_flag = !run_flag; // 切换运行状态
        Send_State_Feedback(); // 串口同步反馈
    }
} else if (HAL_GPIO_ReadPin(KEY0_GPIO_Port, KEY0_Pin) == GPIO_PIN_SET) {
    key0_time = HAL_GetTick();
    key0_lock = 0;
}

