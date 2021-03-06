# 快速傅立葉變換加速器 (FFT)

## 概述

FFT 模組是用硬體的方式來實現 FFT 的基 2 時分運算加速。

## 功能描述

目前該模組可以支持64點、128點、256點以及512點的 FFT 以及 IFFT。在 FFT 內部有兩塊大小為 512 * 32 bit 的 SRAM，在配置完成後 FFT 會向 DMA 發送 TX 請求，將 DMA 送來的送據放到其中的一塊 SRAM 中去，直到滿足當前 FFT 運算所需要的資料量並開始 FFT 運算，蝶形運算單元從包含有有效資料的 SRAM 中讀出資料，運算結束後將資料寫到另外一塊 SRAM 中去，下次蝶形運算再從剛寫入的 SRAM 中讀出資料，運算結束後並寫入另外一塊 SRAM，如此反覆交替進行直到完成整個 FFT 運算。

## API參考

對應的頭文件 `fft.h`

為用戶提供以下介面：

- [fft\_complex\_uint16](#fftcomplexuint16)

### fft\_complex\_uint16

#### 描述

FFT 運算。

#### 函數原型

```c
void fft_complex_uint16(uint16_t shift, fft_direction_t direction, const uint64_t *input, size_t point_num, uint64_t *output);
```

#### 參數

| 參數名稱     |   描述                     |  輸入輸出  |
| --------   | ---------------             | ----     |
| shift    | FFT模組16位寄存器導致資料溢出(-32768~32767)，FFT變換有9層，shift決定哪一層需要移位操作(如0x1ff表示9層都做移位操作；0x03表示第第一層與第二層做移位操作)，防止溢出。如果移位了，則變換後的幅值不是正常FFT變換的幅值，對應關係可以參考fft_test測試demo程序。包含了求解頻率點、相位、幅值的示例| 輸入 |
| direction | FFT 正變換或是逆變換              | 輸入 |
| input | 輸入的資料序列，格式為 RIRI.., 實部與虛部的精度都為 16 bit | 輸入|
| point_num | 待運算的資料點數，只能為 512/256/128/64 | 輸入 |
| output | 運算後結果。格式為 RIRI.. ,實部與虛部的精度都為 16 bit | 輸出 |

### 舉例

```c
#define FFT_N               512U
#define FFT_FORWARD_SHIFT   0x0U
#define FFT_BACKWARD_SHIFT  0x1ffU
#define PI                  3.14159265358979323846
for (i = 0; i < FFT_N; i++)
{
    tempf1[0] = 0.3 * cosf(2 * PI * i / FFT_N + PI / 3) * 256;
    tempf1[1] = 0.1 * cosf(16 * 2 * PI * i / FFT_N - PI / 9) * 256;
    tempf1[2] = 0.5 * cosf((19 * 2 * PI * i / FFT_N) + PI / 6) * 256;
    data_hard[i].real = (int16_t)(tempf1[0] + tempf1[1] + tempf1[2] + 10);
    data_hard[i].imag = (int16_t)0;
}
for (int i = 0; i < FFT_N / 2; ++i)
{
    input_data = (fft_data_t *)&buffer_input[i];
    input_data->R1 = data_hard[2 * i].real;
    input_data->I1 = data_hard[2 * i].imag;
    input_data->R2 = data_hard[2 * i + 1].real;
    input_data->I2 = data_hard[2 * i + 1].imag;
}
fft_complex_uint16(FFT_FORWARD_SHIFT, FFT_DIR_FORWARD, buffer_input, FFT_N, buffer_output);
for (i = 0; i < FFT_N / 2; i++)
{
    output_data = (fft_data_t*)&buffer_output[i];
    data_hard[2 * i].imag = output_data->I1 ;
    data_hard[2 * i].real = output_data->R1 ;
    data_hard[2 * i + 1].imag = output_data->I2 ;
    data_hard[2 * i + 1].real = output_data->R2 ;
}
for (int i = 0; i < FFT_N / 2; ++i)
{
    input_data = (fft_data_t *)&buffer_input[i];
    input_data->R1 = data_hard[2 * i].real;
    input_data->I1 = data_hard[2 * i].imag;
    input_data->R2 = data_hard[2 * i + 1].real;
    input_data->I2 = data_hard[2 * i + 1].imag;
}
fft_complex_uint16(FFT_BACKWARD_SHIFT, FFT_DIR_BACKWARD, buffer_input, FFT_N, buffer_output);
for (i = 0; i < FFT_N / 2; i++)
{
    output_data = (fft_data_t*)&buffer_output[i];
    data_hard[2 * i].imag = output_data->I1 ;
    data_hard[2 * i].real = output_data->R1 ;
    data_hard[2 * i + 1].imag = output_data->I2 ;
    data_hard[2 * i + 1].real = output_data->R2 ;
}
```

## 資料類型

相關資料類型、資料結構定義如下：

- [fft\_data\_t](#fftdatat)：FFT 運算傳入的資料格式。
- [fft\_direction\_t](#fftdirectiont)：FFT 運算模式。

### fft\_data\_t

#### 描述

FFT運算傳入的資料格式。

#### 定義

```c
typedef struct tag_fft_data
{
    int16_t I1;
    int16_t R1;
    int16_t I2;
    int16_t R2;
} fft_data_t;
```

#### 成員

| 成員名稱 | 描述 |
| ----- | --- |
| I1 | 第一個資料的虛部  |
| R1 | 第一個資料的實部  |
| I2 | 第二個資料的虛部  |
| R2 | 第二個資料的實部  |

### fft\_direction\_t

#### 描述

FFT運算模式

#### 定義

```c
typedef enum tag_fft_direction
{
    FFT_DIR_BACKWARD,
    FFT_DIR_FORWARD,
    FFT_DIR_MAX,
} fft_direction_t;
```

#### 成員

| 成員名稱 | 描述 |
| ----- | --- |
| FFT\_DIR\_BACKWARD | FFT 逆變換 |
| FFT\_DIR\_FORWARD  | FFT 正變換 |