# Liblily 

## 简介

`liblily`是一个用于音频处理python库，可以实现音乐的加噪去噪，音乐的节拍识别与分割，音高识别的功能。
同时，为了方便日常使用，本库提供了用于交互的命令行工具。

```python
# 加噪
liblily.noise

# 去噪
liblily.filter

# 节拍识别
liblily.recoginze.tempo_recognize(y, sr)

# 音高识别
liblily.recoginze.freq_recognize(y, sr, ...)
```

## 加噪与去噪

TODO:

## 节拍识别

TODO:

## 音高识别
### 常见算法
- 并行处理法

    音乐信号在经过预处理后由峰值处理器找出峰点和谷点，产生6个脉序列，由6个并行的基音周期估计器估计基音周期。最后对这些基音估计器的输出作逻辑组合，得出估计值。

    缺点：**误差较大**，甚至会大于100 音分。

- 谐波峰值法

    基于快速傅里叶变换的分析法，将信号通过FFT 变换得到离散的频率谱， 最大峰值对应于基音频率。
    
    缺点：如果乐器谐波比较丰富，很有可能把**二次甚至三次谐波误定为音高**。

- 小波分析法

    小波具有良好的时频特性，能很好地调节时域和频域的分辨率。做基音检测时，小波变换相当于一个中心频率和带宽可调的滤波器，每经一次变换，高频谐波部分就被滤去一半，而基音部分被保存下来，变换后的波形也越来越“ 纯”，最终可确定基音。

    缺点：小波分解的运算量很大，尤其当需要较大的分解尺度时。


### 简介

对于一段音乐，首先使用节拍检测对音符进行分割。并对分割后的音符片段使用以下算法进行音高识别。

先用**时域处理法**对音乐信号进行**基音频率**估计， 然后以得到的基音频率为参数设计**数字低通滤波器**。音乐信号经滤波器滤波后，滤除了高频分量，相当于减小了音乐信号的频宽， 排除了谐波成分的干扰。在此基础上再做**FFT** 就可得到准确的音高。

![process](https://github.com/ethan-iai/liblily/blob/master/images/process.png)

### 时域法估计基音频率

时域法估计基音频率采用自相关处理法。
计算待检测信号的自相关函数𝑅_𝑛(𝑘)。一般当𝑹_𝒏(𝒌)为最大值时的k 值就是基音周期。
但由于谐波峰值的干扰，很有可能𝑅_𝑛(𝑘)为最大值时的*k*值为谐波周期，这样就会将谐波频率误认为基音频率。为了防止这种基音误取情况的发生，可对自相关系数𝑅_𝑛(𝑘)再进行一次自相关处理，此时得到的自相关系数𝑅_𝑛′(𝑘′)为最大值时的k′ 值即为基音周期。
基音频率估值可由𝑓=𝑠𝑟/𝑘计算，*sr*为采样频率(sample rate)。

处理`demo.wav`的自相关函数如下如图所示:

![autocorrelate](https://github.com/ethan-iai/liblily/blob/master/images/autocorrelate.jpg)

### 降低处理时间：三中心削波

采用三电平中心削波，将音乐信号的取值限制为-1，0，1三种情况：

- 对于绝对值大于削波阈值的正信号取值为1
- 对于绝对值大于削波阈值的负信号取值为-1
- 对于绝对值小于削波阈值的信号取值为0

通过此操作，将复杂的自相关函数的计算转换为较为简单的逻辑组合，从而提升计算速度。

通常，削波阈值*thres = partial * 音频信号波峰的均值* ，*partial*为一经验常数，此例中取值为0.8。下图中，红色纵线代表峰值位置，黄色横线代表阈值取值。

![slice](https://github.com/ethan-iai/liblily/blob/master/images/slice.jpg)

### 设计滤波器与滤波
当得到基音频率估值𝑓_0′后，就需要以𝑓_0′为参数来设计数字低通滤波器。设计滤波器的目的是为了在**排除高频谐波分量的干扰**和**减小音乐信号的频宽**的基础上用FFT来准确定位基音频率。
在实际处理中，我们选取了𝑓_0′为通带频率， 𝑓_0′为阻带频率，通带衰减3dB、阻带衰减20dB为技术指标的数字低通滤波器，即4阶𝜔_c=𝑓_0′的*butterworth*数字滤波器。

下图从上至下分别展示了滤波器响应，`demo.wav`滤波前的信号频谱图，`demo.wav`滤波后的信号频谱图。

![filter](https://github.com/ethan-iai/liblily/blob/master/images/filter.jpg)

### FFT

音乐信号经滤波后，通过**FFT**后就可得到无谐波干扰的理想频谱，**最大峰值**即为基音频率𝑓_0，从而得到准确的基音频率。

下图比较了经过滤波前后使用谐波峰值法求取音高（基音频率）的效果展示。左图（滤波前）中，音乐信号平款较大，谐波较为丰富，对基音识别造成了干扰；右图（滤波后），排除了谐波的干扰，得出了较为准确的基音频率。

![contrast](https://github.com/ethan-iai/liblily/blob/master/images/contrast.jpg)

### 结果分析

本报告以舒伯特的小夜曲前10s的片段为例,对算法进行介绍。

![demo](https://github.com/ethan-iai/liblily/blob/master/images/demo.png)

对于第一个音符，本算法的识别结果为*147.34Hz*，即*D3*。识别结果与实际音符一致，并且音高（基因频率）识别的误差在1.0%以内。（*D3*的实际频率为*146.83Hz*）

与此同时，中心削波的使用降低了计算的复杂度，提高了处理效率。
## 下载与安装
```shell
mkdir ~/.liblily
git clone https://github.com/ethan-iai/liblily.git
cd liblily
pip install -r requirements.txt
pip install -e .
```

## 使用
对`demo.wav`添加**高斯噪声**，并去除添加的噪声，将结果保存到指定目录。同时，输出对`demo.wav`的节拍(bpm)与音高的检测结果。通常，去噪后的声音样本会被保存到`~/.liblily`，可以通过修改`LIBLILY_SRC_DIR`环境变量改变保存结果的目录。

```shell
python -m liblily.cli examples/demo.wav --noise guass 
```

更进一步，你可以通过命令行设定算法的**参数**。

```shell
python -m liblily.cli examples/demo.wav --noise guass --partial 0.8
```

### 高级
调节算法参数时，可以添加`--verbose`选项，使程序绘制特定图像从而可视化参数的调节情况。

```shell
python -m liblily.cli examples/demo.wav --noise guass --partial 0.8 --verbose
```
