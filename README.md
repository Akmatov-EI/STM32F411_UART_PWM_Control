# STM32F411_UART_PWM_Control
Изначальная задумка: управлять шим сигналом(частатой, импульсом) через UART, всего должно было быть задействовано 4 канала таймера. Uart обрабатывается по прерыванию, а ШИМ должен был отправляться через DMA.
Итоговый вариант: задействовано 4 канала таймера, есть функция управления скважностью шим и его частатой, но инициализация и управление реализовано с CIMSIS так как HAL ограничивал возможности таймера. Есть функция обработки по прерыванию UART где очищается буфер в который изначально были записаны отправленные данные. DMA не был реализован, также не закончена логика меню по переходу и управления ШИМ.

#include "stm32f4xx.h"
#include "main.h"
#include <stdio.h>
#include <stdlib.h>
uint8_t x = 0;
int num = 0;
uint8_t help[10] = "100";
uint8_t buffer[10];
UART_HandleTypeDef huart2;

// Таймеры
uint32_t TimerForTackt = 0;

#define TIM2_CLK 16000000 // Тактовая частота TIM2

void SystemClock_ConfigTimer(void)
{
  // Включаем тактирование для TIM2
  RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;
  // Включаем тактирование для GPIOA
  RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;

  // Настраиваем PA5, PA6, PA7, PA8 на альтернативную функцию (TIM2_CH1, TIM2_CH2, TIM2_CH3, TIM2_CH4)
  GPIOA->MODER |= (2 << (5 * 2)) | (2 << (6 * 2)) | (2 << (7 * 2)) | (2 << (8 * 2));
  GPIOA->OSPEEDR |= (3 << (5 * 2)) | (3 << (6 * 2)) | (3 << (7 * 2)) | (3 << (8 * 2));
  GPIOA->AFR[0] |= (1 << (5 * 4)) | (1 << (6 * 4)) | (1 << (7 * 4)) | (1 << (8 * 4));

  // Настройка TIM2 для ШИМ
  TIM2->PSC = 1600 - 1; // Предделитель (16 МГц / 1600 = 10 кГц)
  TIM2->ARR = 100 - 1;  // Период PWM (10 кГц / 100 = 100 Гц)

  // Инициализация ШИМ для всех 4-х каналов
  TIM2->CCR1 = 10; // Ширина импульса для канала 1
  TIM2->CCR2 = 10; // Ширина импульса для канала 2
  TIM2->CCR3 = 10; // Ширина импульса для канала 3
  TIM2->CCR4 = 10; // Ширина импульса для канала 4

  // Настройка режимов ШИМ
  TIM2->CCMR1 |= (6 << TIM_CCMR1_OC1M_Pos) | (6 << TIM_CCMR1_OC2M_Pos); // PWM mode 1 для каналов 1 и 2
  TIM2->CCMR2 |= (6 << TIM_CCMR2_OC3M_Pos) | (6 << TIM_CCMR2_OC4M_Pos); // PWM mode 1 для каналов 3 и 4

  TIM2->CCMR1 |= TIM_CCMR1_OC1PE | TIM_CCMR1_OC2PE; // Разрешить предобработку
  TIM2->CCMR2 |= TIM_CCMR2_OC3PE | TIM_CCMR2_OC4PE; // Разрешить предобработку

  TIM2->CCER |= TIM_CCER_CC1E | TIM_CCER_CC2E | TIM_CCER_CC3E | TIM_CCER_CC4E; // Включить каналы

  TIM2->CR1 |= TIM_CR1_CEN; // Запуск таймера
}

void SetPWMFrequency(int frequency)
{
  TIM2->PSC = (TIM2_CLK / frequency) - 1; // Обновление предделителя
}

void SetPWMDutyCycle(uint8_t channel, uint8_t dutyCycle)
{
  if (channel < 1 || channel > 4)
    return;                                                    // Защита от выхода за пределы
  uint32_t compareValue = (dutyCycle * (TIM2->ARR + 1)) / 100; // Рассчитываем значение для CCR
  switch (channel)
  {
  case 1:
    TIM2->CCR1 = compareValue;
    break;
  case 2:
    TIM2->CCR2 = compareValue;
    break;
  case 3:
    TIM2->CCR3 = compareValue;
    break;
  case 4:
    TIM2->CCR4 = compareValue;
    break;
  }
}

void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size)
{
  HAL_UART_Transmit_IT(&huart2, buffer, Size);
  HAL_UARTEx_ReceiveToIdle_IT(&huart2, buffer, 10);
  for (int i = 0; i < sizeof(buffer); i++)
  {
    buffer[i] = 0; // Каждый элемент устанавливаем на 0
  }
}

/* Private function prototypes -----------------------------------------------*/
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART2_UART_Init(void);

enum
{
  START_MESSAGE = 0,
  FIRST_CHANNEL = 1,
  SECOND_CHANNEL = 2,
  THIRD_CHANNEL = 3,
  FOURTH_CHANNEL = 4
};

typedef struct
{
  uint8_t Frequance;
  uint8_t PWM_One;
  uint8_t PWM_Two;
  uint8_t PWM_Three;
  uint8_t PWM_Four;

} Commands;
Commands comm;
int main(void)
{

  HAL_Init();

  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART2_UART_Init();
  // MX_TIM4_Init();

  // uint8_t zero[] = {0};
  uint8_t HelpText[] = "Help Text\n";
  uint8_t WrongText[] = "Erorr";
  uint8_t StartText[] = "\rWelcome to the application for setting up a PWM signal:\n basic commands:       help\n";
  uint8_t StartMenu[] = "\rStart Menu, select frequency and channel to work with PWM signal\n";
  comm.Frequance = "frequance";
  // float sensor_value = 1234.56f; // Пример значения с датчика
  HAL_UART_Transmit(&huart2, StartText, sizeof(StartText), HAL_MAX_DELAY);
  HAL_Delay(100);
  SystemClock_ConfigTimer();
  uint8_t step = 0;
  while (1)
  {
    SetPWM(199, num);
    HAL_UARTEx_ReceiveToIdle_IT(&huart2, buffer, 10);
    if (strcmp(buffer, help) == 0)
    { // Сравниваем buffer с help
      num = atoi(buffer);
      HAL_UART_Transmit(&huart2, HelpText, strlen(HelpText), HAL_MAX_DELAY);
      HAL_Delay(100);
    }
    if (strcmp(buffer, WrongText) == 0)
    { // Сравниваем buffer с help
      // x++;
      HAL_UART_Transmit(&huart2, WrongText, strlen(WrongText), HAL_MAX_DELAY);
      HAL_Delay(100);
    }
  }
  switch (step)
  {
  case FIRST_CHANNEL:
    if (HAL_GetTick() - TimerForTackt >= 10000)
    {
      HAL_UART_Transmit(&huart2, StartMenu, strlen(StartMenu), HAL_MAX_DELAY);
      TimerForTackt = HAL_GetTick();
    }
    if (strcmp(buffer, comm.Frequance) == 0)
    { 
      while(num==0)
      {
        num = atoi(buffer);
        SetPWMFrequency(num);
      }
    }
    break;

  default:
    break;
  }
}

void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSI;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV1;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0) != HAL_OK)
  {
    Error_Handler();
  }
}

static void MX_USART2_UART_Init(void)
{

  huart2.Instance = USART2;
  huart2.Init.BaudRate = 115200;
  huart2.Init.WordLength = UART_WORDLENGTH_8B;
  huart2.Init.StopBits = UART_STOPBITS_1;
  huart2.Init.Parity = UART_PARITY_NONE;
  huart2.Init.Mode = UART_MODE_TX_RX;
  huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart2.Init.OverSampling = UART_OVERSAMPLING_16;
  if (HAL_UART_Init(&huart2) != HAL_OK)
  {
    Error_Handler();
  }
  /* USER CODE BEGIN USART2_Init 2 */

  /* USER CODE END USART2_Init 2 */
}

static void MX_GPIO_Init(void)
{
  /* USER CODE BEGIN MX_GPIO_Init_1 */
  /* USER CODE END MX_GPIO_Init_1 */

  /* GPIO Ports Clock Enable */
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOB_CLK_ENABLE();

  /* USER CODE BEGIN MX_GPIO_Init_2 */
  /* USER CODE END MX_GPIO_Init_2 */
}

void Error_Handler(void)
{
  /* USER CODE BEGIN Error_Handler_Debug */
  /* User can add his own implementation to report the HAL error return state */
  __disable_irq();
  while (1)
  {
  }
  /* USER CODE END Error_Handler_Debug */
}

#ifdef USE_FULL_ASSERT
/
    *@brief Reports the name of the source file and the source line number
         *where the assert_param error has occurred.
             *@param file : pointer to the source file name
                                *@param line : assert_param error line source number
                                                   *@retval None
                                                       * /
    void assert_failed(uint8_t *file, uint32_t line)
{
  /* USER CODE BEGIN 6 */
  /* User can add his own implementation to report the file name and line number,
     ex: printf("Wrong parameters value: file %s on line %d\r\n", file, line) */
  /* USER CODE END 6 */
}
#endif /* USE_FULL_ASSERT */
