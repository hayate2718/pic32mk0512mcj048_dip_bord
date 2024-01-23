# pic32mk0512mcj048_dip_bord
## 仕様
1. PIC32MK0512MCJ048のDIP化基板
2. 振動子などの最低限必要な外部回路を搭載する
3. 書き込み端子以外の汎用IOをすべて使用可にする
4. CANトランシーバ搭載、LED搭載
5. メインオシレータ16MHz
6. セカンダリオシレータ32.768KHz

ReservedPin

|Number|Name|Function|
|---|---|---|
|4|MCLR|MCLR|
|5|VSS|VSS|
|6|VDD|VDD|
|11|RB0|**VREF-**|
|12|RB1|**VREF+**|
|15|AVDD|AVDD|
|16|AVSS|AVSS|
|21|VSS|VSS|
|22|VDD|VDD|
|25|RA4|**C1RX**|
|26|VDD|VDD|
|27|RC12|**OSCI**|
|28|RC15|**OSCO**|
|29|VSS|VSS|
|30|RD8|**LED_R**|
|31|RB5|**PGD2**|
|32|RB6|**PGC2**|
|33|RC10|**LED_G**|
|34|RB7|**C1TX**|
|35|RC13|**SOSCI**|
|36|RB8|**SOSCO**|
|42|VSS|VSS|
|43|VDD|VDD|

![pic32mcj048_top](https://github.com/hayate2718/pic32mk0512mcj048_dip_bord/assets/58509900/6c386add-c361-40bc-9075-142109f2b9cd)


## エラッタ
Harmony V3 でコードを出力して、PLIBを使ったときに出力されたコードを書き換えないとうまく動作しないことがあった。<br>
その対処を記録しておく。

### コンパイラの最適化
割り込みハンドラと共有する変数に`volatile`修飾子をつける。

### SPI slave
受信割り込みハンドラ内でSPIxBUFが一回目の読み取り以降変化しない。<br>
対処:SPIxBUFに書き込みを行いSPIxBUFをクリアする。<br>
以下例
```c++
void SPI2_RX_InterruptHandler (void)
{
    uint32_t receivedData = 0;

    spi2Obj.rxInterruptActive = true;

    while (!(SPI2STAT & _SPI2STAT_SPIRBE_MASK))
    {
        /* Receive buffer is not empty. Read the received data. */
        receivedData = SPI2BUF;
        
        memset((void*)&SPI2BUF,0,sizeof(SPI2BUF)); //add code

        if (spi2Obj.rdInIndex < SPI2_READ_BUFFER_SIZE)
        {
            SPI2_ReadBuffer[spi2Obj.rdInIndex++] = receivedData;
        }
    }

    /* Clear the receive interrupt flag */
    SPI2_CLEAR_RX_INT_FLAG();

    spi2Obj.rxInterruptActive = false;

    /* Check if CS interrupt occured before the RX interrupt and that CS interrupt delegated the responsibility to give
     * application callback to the RX interrupt */

    if (spi2Obj.csInterruptPending == true)
    {
        spi2Obj.csInterruptPending = false;
        spi2Obj.transferIsBusy = false;

        spi2Obj.wrOutIndex = 0;
        spi2Obj.nWrBytes = 0;

        if(spi2Obj.callback != NULL)
        {
            spi2Obj.callback(spi2Obj.context);
        }

        /* Clear the read index. Application must read out the data by calling SPI2_Read API in the callback */
        spi2Obj.rdInIndex = 0;
    }
}
```
## 考え中
<b>
PINネームを大きめに入れたい<br>
LED駆動用FETをプルダウンする<br>
IO引き出し線をもう少し太くしたい<br>
在庫があればCANトラをSOICにしたい<br>
redemi が長くなりすぎたらwikiにまとめる。
</b>
