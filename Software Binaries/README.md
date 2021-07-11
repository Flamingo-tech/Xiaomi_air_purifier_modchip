After open sourcing the mod chip I wanted to give some insights on how the mod-chip itself works.

open source info: [click](https://flamingo-tech.nl/2021/07/10/xiaomi-modchip-open-source/)

![](https://flamingo-tech.nl/wp-content/uploads/2021/07/image-34-1024x642.png)

System overview Xiaomi air purifiers (3H/C/PRO)

As can be seen in the image above, the system consist out of several sub systems.  
The two most important sub-systems are:

1.  NFC Filter
2.  Stm32F412RET6

There are two ways to bypass the NFC filters, Make an implant that mimics the behaviour of the NXP NFC IC. Or use Ghidra to reverse engineer the firmware and remove the NFC checks from the software all together.  
I did both (the Ghidra hack we will discuss later)

Pinout of the Xiaomi air purifier 3h/c/pro NFC PCBA connector:  
1.SDA  
2.VDD  
3.PWDOWN  
4.GND  
5.SCL  
Where to find the Xiaomi air purifier 3H/C/Pro NFC PCB: [Click](https://flamingo-tech.nl/2021/06/30/how-to-install-xiaomi-air-purifier-mod-chip-to-your-3h-3c-or-pro-device/)

From an ease of entry standpoint it's just easier to swap out the NFC board than to take the whole thing apart and reflash the ST microcontroller.

Now to the juicy bits, How does it work:  
When you open the filter door or turn on the the air purifier the device will communicate over I2C with the filters.  
Right before first communication is done it will pull the PWDOWN line low.  
so the first piece of the code is (ofc after all the STM32 settings):

![](https://flamingo-tech.nl/wp-content/uploads/2021/07/image-36.png)

```
 if (HAL_GPIO_ReadPin(GPIOA, PDOWN_Pin)==0) { 
     HAL_I2C_EnableListen_IT(&hi2c1);
 }
 else{

     HAL_I2C_DisableListen_IT(&hi2c1);
     rx_counter=0;
}


```

This piece of code will enable the I2C Interrupts to start functioning. Else: Disable the interrupts and set rx_counter to zero.

![](https://flamingo-tech.nl/wp-content/uploads/2021/07/image-37.png)

```
void HAL_I2C_AddrCallback(I2C_HandleTypeDef *hi2c, uint8_t TransferDirection, uint16_t AddrMatchCode)
{
    __HAL_I2C_CLEAR_FLAG(hi2c, I2C_FLAG_ADDR);

	if(TransferDirection == I2C_DIRECTION_RECEIVE) //Need to send
	{

		HAL_I2C_Slave_Seq_Transmit_IT(&hi2c1, TX_COMMAND, 16, I2C_FIRST_AND_LAST_FRAME);
		rx_counter++;

	}
	 else if(TransferDirection == I2C_DIRECTION_TRANSMIT)//Need to receive
	{

		HAL_I2C_Slave_Seq_Receive_IT(&hi2c1, rx_buff, sizeof(rx_buff), I2C_FIRST_FRAME);
		tx_counter++;

	}
```

the rx_counter is important, why: see code above.  
Every time the air purifier sends a message on the I2C bus and PWDON ==0 it will check if we need to send a message or receive a message. If we need to send a message (thus respond to previous messages) I will do rx_counter++ (+1). (I do the same if we need to receive something)  
Then I mapped all the response in a list:

![](https://flamingo-tech.nl/wp-content/uploads/2021/07/image-38.png)

Xiaomi air purifier NFC responses

And when the response counter corresponds with the rx_counter, it will send the appropriate commands.

I can't share the complete code I wrote, since it would be to easy for Xiaomi to bypass the modchip.
