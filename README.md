microc-os
=========

How to use Microc/os 
#include <avr/io.h>
#include <util/delay.h>
#include <avr/interrupt.h>
#include "OSAcfg.h"
#include <OSA.h>
#include "my_keypad.h"
#include "my_LCD.h"

// Message MailBox
OST_MSG_CB    MB_Handler;


void InitT0(void)
{
	// Timer/Counter 0 initialization
    // Clock source: System Clock
    // Clock value: 125.000 kHz
    // Mode: Normal top=0xFF
    // OC0 output: Disconnected
    TCCR0=0x03;
    TCNT0=0x00;
    OCR0=0x00;

	// Timer(s)/Counter(s) Interrupt(s) initialization
    TIMSK=0x01;
}

ISR(TIMER0_OVF_vect)
{
	// Place your code here
	// Transfer HW tick to OSA Kernal
	// To handle the required Delay
    OS_Timer();
}

void Task_LCD(void)
{
	void *value;
	LCD_Init();
	while(1)
	{
		OS_Msg_Wait(MB_Handler, value);
		LCD_SendData(*((unsigned char *)value));
		OS_Delay(100);
	}
}

void Task_KeyPad(void)
{
	unsigned char pressedKey = NO_PRESSED_KEY;
	static unsigned char sKey = NO_PRESSED_KEY;
	KeyPad_Init();
	while(1)
	{
		pressedKey = KeyPad_getKey();
		if(pressedKey != NO_PRESSED_KEY)
		{
			// send it to LCD_Task
			sKey = pressedKey + '0';
			OS_Msg_Send(MB_Handler, (void*)&sKey);
		}
		OS_Delay(1);
	}
}


int main(void)
{
	OS_Init();

  	OS_Task_Create(0, Task_KeyPad);
	OS_Task_Create(1, Task_LCD);
	OS_Msg_Create(MB_Handler);

	// Init T0 OverFlow Interrupt
	InitT0();
	// Enable Global Interrupts
	SREG |= 0x80; 
  	
	OS_Run();
	return 0;
}
