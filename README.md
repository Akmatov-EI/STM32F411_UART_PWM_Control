# STM32F411_UART_PWM_Control
Изначальная задумка: управлять шим сигналом(частатой, импульсом) через UART, всего должно было быть задействовано 4 канала таймера. Uart обрабатывается по прерыванию, а ШИМ должен был отправляться через DMA.
Итоговый вариант: задействовано 4 канала таймера, есть функция управления скважностью шим и его частатой, но инициализация и управление реализовано с CIMSIS так как HAL ограничивал возможности таймера. Есть функция обработки по прерыванию UART где очищается буфер в который изначально были записаны отправленные данные. DMA не был реализован, также не закончена логика меню по переходу и управления ШИМ.
