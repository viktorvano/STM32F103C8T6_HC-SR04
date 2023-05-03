# STM32F103C8T6_HC-SR04
 Multiple ultrasonic sensors HC-SR04 example  
 Video: https://youtu.be/Jmjtri25QWo  
 
 
## Wiring Diagram
![alt text](https://github.com/viktorvano/STM32F103C8T6_HC-SR04/blob/main/documentation%20files/wiring%20diagram.png?raw=true)  
  
![alt text](https://raw.githubusercontent.com/viktorvano/STM32-Bootloader/master/images/STM32F103C8T6_Bottom.jpg?raw=true)  
  
  
## Calculations
  
Speed of sound at 22 °C is 344 m/s.  
Sound travels 1 cm one way in just 29 us. Calculation: 0.01m / 344m/s = 29 us.  
But the sound travels from the sensor, hits the obstacle and goes back, therefore each centimeter takes twice that much 29 us * 2 = 58us.  
  
  
## Clock, Pinout and Timer Settings
  
![alt text](https://github.com/viktorvano/STM32F103C8T6_HC-SR04/blob/main/documentation%20files/pinout.png?raw=true)  
  
![alt text](https://github.com/viktorvano/STM32F103C8T6_HC-SR04/blob/main/documentation%20files/clock%20settings.png?raw=true)  
  
![alt text](https://github.com/viktorvano/STM32F103C8T6_HC-SR04/blob/main/documentation%20files/timer2.png?raw=true)  
  
  
## Code Snippets
  
main.c  
```C
  /* USER CODE BEGIN 2 */
  HAL_Delay(500);
  HAL_GPIO_WritePin(USB_Enable_GPIO_Port, USB_Enable_Pin, GPIO_PIN_SET);

  char message[200], number[8];
  uint16_t distancesInCm[3];
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
	  HAL_Delay(900);
	  CDC_Transmit_FS((uint8_t*)&"\n\n\nMeasuring Distances...\n", strlen("\n\n\nMeasuring Distances...\n"));
	  HAL_Delay(100);
	  memset(message, 0, 200);
	  for(uint16_t i=0; i<3; i++)
	  {
		  sensor = i;//update sensor index in timer2
		  distancesInCm[i] = measureDistance(triggerPorts[i], triggerPins[i], echoPorts[i], echoPins[i]);
		  memset(number, 0, 8);
		  itoa(distancesInCm[i], number, 10);
		  strcat(message, number);
		  strcat(message, " cm\n");
	  }
	  CDC_Transmit_FS((uint8_t*)&message, strlen(message));
  }
  /* USER CODE END 3 */
```
  
myLibrary.h  
```C
#ifndef INC_MYLIBRARY_H_
#define INC_MYLIBRARY_H_

#include "main.h"
#include "usb_device.h"
#include "usbd_cdc_if.h"

extern TIM_HandleTypeDef htim2;

#define TriggerDuration 2

uint16_t distance, triggerTime, sensor;
GPIO_TypeDef *triggerPorts[3];
uint16_t triggerPins[3];
GPIO_TypeDef *echoPorts[3];
uint16_t echoPins[3];

void SysTickEnable();
void SysTickDisable();
uint16_t measureDistance(GPIO_TypeDef *triggerPort, uint16_t triggerPin, GPIO_TypeDef *echoPort, uint16_t echoPin);

#endif /* INC_MYLIBRARY_H_ */
```
  
myLibrary.c  
```C
#include "myLibrary.h"

uint16_t distance = 0, triggerTime = 0, sensor = 0;
GPIO_TypeDef *triggerPorts[3] = {Trigger0_GPIO_Port, Trigger1_GPIO_Port, Trigger2_GPIO_Port};
uint16_t triggerPins[3] = {Trigger0_Pin, Trigger1_Pin, Trigger2_Pin};
GPIO_TypeDef *echoPorts[3] = {Echo0_GPIO_Port, Echo1_GPIO_Port, Echo2_GPIO_Port};
uint16_t echoPins[3] = {Echo0_Pin, Echo1_Pin, Echo2_Pin};

void SysTickEnable()
{
	__disable_irq();
	SysTick->CTRL |= (SysTick_CTRL_CLKSOURCE_Msk | SysTick_CTRL_ENABLE_Msk);
	__enable_irq();
}

void SysTickDisable()
{
	__disable_irq();
	SysTick->CTRL &= ~(SysTick_CTRL_CLKSOURCE_Msk | SysTick_CTRL_ENABLE_Msk);
	__enable_irq();
}

uint16_t measureDistance(GPIO_TypeDef *triggerPort, uint16_t triggerPin, GPIO_TypeDef *echoPort, uint16_t echoPin)
{
	if(!HAL_GPIO_ReadPin(echoPort, echoPin))//skip sensor if ECHO pin is still busy
	{
		SysTickDisable();
		HAL_TIM_Base_Start_IT(&htim2);
		HAL_GPIO_WritePin(triggerPort, triggerPin, GPIO_PIN_SET);
		triggerTime = 0;//reset the variable
		asm ("nop");//to avoid program freezing
		while(triggerTime < TriggerDuration);
		HAL_GPIO_WritePin(triggerPort, triggerPin, GPIO_PIN_RESET);
		while(!HAL_GPIO_ReadPin(echoPort, echoPin));
		distance = 0;//reset the variable
		while(HAL_GPIO_ReadPin(echoPort, echoPin));
		HAL_TIM_Base_Stop_IT(&htim2);
		SysTickEnable();
	}else//give max distance if ECHO pin is still busy
	{
		distance = 500;
	}
	return distance;
}
```
  
stm32f1xx_it.c  
```C
void TIM2_IRQHandler(void)
{
  /* USER CODE BEGIN TIM2_IRQn 0 */
	if(HAL_GPIO_ReadPin(echoPorts[sensor], echoPins[sensor]) && distance < 500)
		distance++;
	if(triggerTime < TriggerDuration)
		triggerTime++;
  /* USER CODE END TIM2_IRQn 0 */
  HAL_TIM_IRQHandler(&htim2);
  /* USER CODE BEGIN TIM2_IRQn 1 */

  /* USER CODE END TIM2_IRQn 1 */
}
```
