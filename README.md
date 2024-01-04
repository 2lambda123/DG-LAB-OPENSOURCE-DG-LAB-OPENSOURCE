# DG-LAB 开源
DG-LAB设备在全球范围得到广大朋友的认可与喜爱.很多朋友们希望我们的设备可以参与到更多的场景中去,为此我们将DG-LAB 具有代表性的设备蓝牙协议以开源的形式分享出来，您可以通过无数种其他编程的方式将DG-LAB设备参与到您自己的娱乐场景中去.

> 开源蓝牙协议旨在让DG-LAB爱好者更加自由的使用设备，未经授权请勿将本内容用以任何商业用途,如有需要,请[联系我们](https://www.dungeon-lab.com)

![郊狼电击器LOGO](image/DG-LAB_492.png)

## 郊狼情趣电击器

### 郊狼情趣电击器
|    服务UUID    |    特性UUID     |      属性      |      名称      |    大小(BYTE)  |
| :------------: | :------------: | :------------: | :------------: | :------------: |
|     0x180A     |     0x1500     |    读/通知     | Battery_Level  | 1字节           |
|     0x180B     |     0x1504     |    读/写/通知  | PWM_AB2        | 3字节           |
|     0x180B     |     0x1506     |    读/写       | PWM_A34        | 3字节           |
|     0x180B     |     0x1505     |    读/写       | PWM_B34        | 3字节           |

|      名称      |      说明       |  通信数据定义  |
| :------------: | :------------: | :------------: |
| Battery_Level  | 设备当前电量    | 1byte(整数0-100)|
| PWM_AB2        | AB两通道强度    | 23-22bit(保留)	21-11bit(B通道实际强度)	10-0bit(A通道实际强度)  |
| PWM_A34        | A通道波形数据   | 23-20bit(保留)	19-15bit(Az)	14-5bit(Ay)	4-0bit(Ax)  |
| PWM_B34        | B通道波形数据   | 23-20bit(保留)	19-15bit(Bz)	14-5bit(By)	4-0bit(Bx)  |

> 基础UUID: 955A`xxxx`-0FE2-F5AA-A094-84B8D4F3E8AD (将xxxx替换为服务的UUID)

### 基本原理
郊狼内置了两组独立的脉冲生成模块，分别对应A，B两个通道。每组脉冲生成模块又由电源模块和波形控制模块两部分构成。我们通过蓝牙协议中的S,X,Y,Z四个变量来控制脉冲生成模块

### 电源模块(S)
> S: PWM_AB2特性

电源模块控制脉冲的电压，也就是通道的强度（App界面中圆环内的数字)，对应蓝牙协议中的参数S，范围是【0-2047】（不能超过2047）。在我们的APP中每增加一点强度是增加7(电击器中设置的实际强度值为APP中显示值的7倍)。当我们向APP内写入不同的参数S的值时，通道强度会立刻改变并且一直保持。

这里给出，`A通道与B通道的强度值`与`Qt字节数据QByteArray`相互转换的C++函数（可能会有bug，仅供参考）：

```
// 根据A通道和B通道的强度值编码得到待发送的字节数据
QByteArray IntensityEncode(int intensityA, int intensityB)
{
	intensityA *= 7;
	intensityB *= 7;
	intensityA = max(0, min(intensityA, 2047));
	intensityB = max(0, min(intensityB, 2047));

	const unsigned char part3 = intensityA >> 5;
	const unsigned char A_Lower = intensityA & 0x1F;
	const unsigned char B_Higher = intensityB >> 8;
	const unsigned char part1 = intensityB & 0xFF;
	const unsigned char part2 = (A_Lower << 3) | B_Higher;

	QByteArray data;
	data.append(part1);
	data.append(part2);
	data.append(part3);

	return data;
}

// 根据字节数据解码得到A通道和B通道的强度值
void IntensityDecode(const QByteArray& data, int& intensityA, int& intensityB)
{
	const unsigned char part1 = data.at(0);
	const unsigned char part2 = data.at(1);
	const unsigned char part3 = data.at(2);

	
	const unsigned char A_Higher = part3 & 0x3F;
	const unsigned char A_Lower = (part2 >> 3) & 0x1F;
	const unsigned char B_Higher = part2 & 0x07;

	intensityA = (A_Higher << 5) | A_Lower;
	intensityB = (B_Higher << 8) | part1;

	intensityA /= 7;
	intensityB /= 7;
}
```

### 波形控制模块(X Y Z)
> X: PWM_A34或PWM_B34中4-0bit的5bits数据<br/>
> Y: PWM_A34或PWM_B34中14-5的10bits数据<br/>
> Z: PWM_A34或PWM_B34中19-15的5bits数据<br/>

波形控制模块控制脉冲出现的规律以及脉冲宽度的变化。脉冲出现的规律和脉冲宽度的变化被以内置波形或者自定义波形的形式保存下来。

### 脉冲规律控制
郊狼的程序把每一秒分割成1000毫秒，在每个毫秒内都可以产生一次脉冲。我们使用蓝牙协议中的X,Y两个参数来编码脉冲产生的规律，其中X代表连续X毫秒发出X个脉冲，Y表示X个脉冲过后会间隔Y毫秒再发出X个脉冲并循环。X的范围是【0-31】，Y的范围是【0-1023】

> e.g.<br/>
参数【1,9】代表每隔9ms发出1个脉冲，总共耗时10ms，也就是脉冲频率为100hz。参数【5,95】代表每隔95ms发出5个脉冲，总共耗时100ms，由于这五个脉冲连在一起并且持续时间仅为5ms，因此使用者只会感受到一次（五合一）脉冲，因此在使用者的体感中脉冲频率为10hz

### Frequency值
```
Frequency = X + Y
脉冲真实频率 = Frequency / 1000
```
作为一个X，Y值互相之间关系的频率特征值。您可以通过设置Frequency值来计算出最合适的X，Y值。其取值范围(10~1000)

X,Y的数据比例保持按照公式:
<div id="gongshi"><pre>
X = ((Frequency / 1000)^ 0.5) * 15
Y = Frequency - X
</pre></div>

此时效果是最好的。

如果 X:Y 的比例大于1:9（例如【8,2】），波形的整体感受会变弱.

### 脉冲宽度控制
一次脉冲由两个对称的正负单极性脉冲组成，两个单极性脉冲的高度(电压)由这个通道的强度决定。我们通过控制脉冲宽度的方式控制脉冲带来的感受的强弱。脉冲越宽，感受就越强，反之脉冲越窄感受约越弱。脉冲宽度的节奏性变化可以创造出不同的脉冲感受。

脉冲宽度由参数Z控制，Z的范围是【0-31】，实际的脉冲宽度为Z×5us。也就是当Z=20时，脉冲宽度为5×20us=100us

- Tips: 当脉冲宽度大于100us（Z>20）时脉冲更容易引起刺痛

这里给出，`X、Y、Z`与`Qt字节数据QByteArray`相互转换的C++函数（可能会有bug，仅供参考）：

```
// 根据X、Y和Z编码得到待发送的字节数据
QByteArray FreqEncode(unsigned char X, unsigned int Y, unsigned char Z)
{
	Z = min(Z, 31);
	const unsigned int part1_higher = Y & 0x7;
	const unsigned int part2_lower = (Y >> 3) & 0x7F;

	const unsigned int part2_higher = Z & 0x1;
	const unsigned int part3_lower = (Z >> 1) & 0xF;

	const unsigned char part1 = (part1_higher << 5) | X;
	const unsigned char part2 = (part2_higher << 7) | part2_lower;
	const unsigned char part3 = part3_lower;

	QByteArray data;
	data.append(part1);
	data.append(part2);
	data.append(part3);

	return data;
}

// 根据字节数据解码得到X、Y和Z
void FreqDecode(const QByteArray& data, unsigned char& X, unsigned int& Y, unsigned char& Z)
{
	const unsigned char part1 = data.at(0);
	const unsigned char part2 = data.at(1);
	const unsigned char part3 = data.at(2);

	X = part1 & 0x1F;
	unsigned int part1_higher = (part1 >> 5) & 0x7;

	unsigned int part2_higher = (part2 >> 7) & 0x1;
	unsigned int part2_lower = part2 & 0x7F;

	unsigned int part3_lower = part3 & 0xF;

	Y = (part2_lower << 3) | part1_higher;
	Z = (part3_lower << 1) | part2_higher;
}
```


### 创造不断变化的波形
由于波形的参数并非固定不变而是在不断变化的。因此在郊狼的设计中每一组【X,Y,Z】参数仅在0.1S时间内有效。也就是说，每当你向设备写入一组【X,Y,Z】参数，设备都会输出参数对应的0.1S波形之后停止输出。也就是说如果你需要波形保持频率100hz，宽度100us并且持续输出，那么你需要每隔0.1秒向设备发送参数【1,9,20】

- Tips: 您也可以每隔0.1秒修改Frequency的值，利用[Frequency公式](#gongshi)来自动生成X，Y的值

### 更多例子
如果你希望创造出一个频率不断变快的波形，可以尝试每0.1秒按照顺序向设备发送如下数据。

【5,135,20】【5,125,20】【5,115,20】【5,105,20】【5,95,20】【4,86,20】【4,76,20】【4,66,20】【3,57,20】【3,47,20】【3,37,20】【2,28,20】【2,18,20】【1,14,20】【1,9,20】

如果你希望创造一个在两个频率之间不断切换的波形，可以尝试每0.1秒按照顺序向设备发送如下数据。

【5,95,20】【5,95,20】【5,95,20】【5,95,20】【5,95,20】【1,9,20】【1,9,20】【1,9,20】【1,9,20】【1,9,20】

如果你希望创造一个频率不变，但是带来“推力”感受的波形，可以尝试每0.1秒按照顺序向设备发送如下数据。

【1,9,4】【1,9,8】【1,9,12】【1,9,16】【1,9,18】【1,9,19】【1,9,20】【1,9,0】【1,9,0】【1,9,0】

- Tips: 人体对频率变化的感受较慢，因此如果频率变化过快将无法形成节奏感。而脉冲宽度的频繁变化则可以创造出多样的感觉
