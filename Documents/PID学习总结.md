# PID学习记录

## 基础驱动代码

- OLED测试
- LED测试
- 定时中断和非阻塞式按键测试（即使按下按键不放，也不会阻止定时器自增）
- 电位器旋钮测试
- 电机测试
- 编码器测试（定时周期读取编码器增量，这个值是Speed，增量值累加后得到的值就是Location）
- 串口测试

## 位置式PID定速控制

**PID闭环控制最终的任务**

PID闭环控制最终的任务是，按键控制目标速度值增减，PID调控，使得编码器测量得到的实际速度和设置的目标速度保持一致。且，即使负载变化，实际速度也都能很好地维持在目标速度附近。

目标速度的范围不能超过电机测试中的实际速度，而且为了使PID有额外的调控余量，因为这个最大范围是空载时候测得的，在实际应用中，电机大概到不了这个速度。

## 增量式PID定速控制

**增量式PID和位置式PID的区别**

增量式PID一旦调好了之后，把Kp，Ki，Kd都调成0，那么Out+=0，只会是Out不再变化，而不会导致Out为0，所以会正常运行。

总结：当增量式PID暂停控制时，Out可以维持在当前值，当增量式PID启动控制时，它也会在当前Out的基础上，继续执行控制。所以，增量时候PID适合自动控制和手动控制切换的场景，比如目前三个参数都给0，相当于PID算法暂停了，此时再改变目标值，输出值不会变，实际值也不会进行跟踪操作，这样相当于进入了手动控制模式。用户可以自己凭柑橘修改Out值，进行手动控制，当修改完Out值后，再启动PID控制，就相当于回到自动控制模式。由手动切换到自动后，此时最能体现出位置式PID和增量式PID的不同。位置式PID，由于内部有个单独的积分变量ErrorInt，所以刚切换到位置式PID控制后，这个积分变量需要进行变化，过一段时间后，才能适应当前状态，使输出值稳定。因此位置式PID，由手动控制切换到自动控制后，系统会由短暂的抖动，过渡不平稳。而增量式PID，由手动切换到自动后，它会在手动修改Out的基础上，进行+=的赋值，又因为增量式PID内部没有单独的积分变量，所以增量式PID在切换过程中可以保证输出值的稳定过渡，因此增量式PID更适合自动控制和手动控制切换的场景。

## 位置式PID定位置控制

在改变目标位置的瞬间，绿线才有调控力度，当实际位置和目标位置重合时，绿线输出力度就归零了。也就是说只有需要改变位置时才需要调控力，当位置停下来固定了，自然不需要持续的力来维持这个位置了。当位置固定后，用手强行改变位置，绿线输出的调控力，也会随之变化，这个力会驱动实际位置向目标位置靠近，在松手后，实际位置会立刻跟踪到目标位置，目标位置和实际位置重合后，调控力也会随之归零。

**定位置控制为什么没有稳态误差**

稳态误差产生的原因是：定速控制，当输出值为0时，实际值还会自发地偏移，但是定位置控制，输出值为0时，电机的位置时不会自发偏移的。

**实际值和目标值为什么会有一点点误差**

当输出的驱动力比较小时，电机可能转不起来，改变不了位置，所以最后一点点误差就无法消除。加入积分项Ki=0.05，可以重合了，但是调控时会超调。（定位置加入I项后必定超调）。定速：要保持定速，Ki用来抵消f；可是定位置不需要输出力来保持位置恒定，所以Ki积累的力必然要输出，超过目标后，误差为负，P项立刻输出反向调控力，而I项仍然会输出正向调控力，只不过，现在开始反向累积，用于抵消之前累积过大的正向驱动力，当正反积分抵消后，实际值重新和目标值重合，此时，P项输出0，I项经过一正一反的累积，抵消了，也输出0，这样最终的位置才能固定下来。

所以面对实际值无法与目标值重合这个问题，可以采用积分分离：如果误差比较小（手动），使用PID控制器，如果误差比较大（调控），使用PD控制器，把I项分离出去，不用了。

Ki=0.05，有点超调，可以改成0.02，超调就好多了，有积分效果，同时最后的位置恢复过程时比较平缓的。

加入积分分离后，现在Ki=0.05，后期的手动旋转有了Ki的调控，但前期调控误差大时不会再出现超调，注意可能会显示重合，多调多转转就好了。

**阈值**

如果阈值过大，那积分分离就没有作用了，如果值过小，那用手改变位置时，一旦误差超过了预支了，积分作用就会消失，失去了积分来对抗外力的效果。

误差正好超过阈值停下来了，积分效果就会瞬间消失。

所以解决办法就是加大阈值（积分分离）、变速积分。

变速积分是在积分分离基础上进行的，因为这个突变过程是不太好的。

##增量式PID定位置控制

增量式PID需要依赖上一次的Out，这个特性带来的好处是方便自动控制和手动控制平滑的切换，但是，这样的特性也有坏处。就是依赖上一次Out，有点违背了P向与历史无关这个设计需求，另外，如果上一次Out是错误的，那么增量式PID的后续调节就会持续受到这个错误影响。但反观位置式PID，每次的Out都是独立的，彼此没有关联，就不会有这个问题。增量式PID，虽然对积分项的包容性很好，但是同时它也非常依赖这个积分项，没有积分项的增量式PID，容易出现实际值与目标值偏移的问题，或者说是，如果上一次的Out值是错误的，同时后续的Error变化很小，则纯比例项很难纠正这个错误。另外，增量式的纯P项调节，需要借助上一次的Out，则会使的即使是纯P项调节，它也会有一点点积分项的特性，所以，增量式PID最好不要给Ki设置为0。

位置式PID是有ErrorInt进行误差积分的累加的，这个单独的积分，其实经常会积过头，造成积分饱和，所以一般需要对积分进行限制，比如积分限幅。增量式PID也有积分过程，这个积分变量是由Out来充当的，Out兼具积分功能后，下面的输出限幅，顺手就限制了积分的幅度，所以，增量式PID，一般不会受到积分饱和问题的困扰。

## 位置式PID定位置控制-积分限幅

```c
//执行PID调控
Actual = Encoder_Get_Filtered();
//获取本次误差和上次误差
Error1 = Error0;
Error0 = Target-Actual;
//计算累计的误差积分
/*有了积分限幅后，这个判断就不需要了，因为这里判断的作用，也是防止积分深度饱和，比如Ki给0，误差始终存在，积分就会深度饱和。
if(fabs(Ki)>EPSILON)
{
ErrorInt+=Error0;
}else
{
ErrorInt = 0;
}*/
			
ErrorInt += Error0;	//注意啊，不要用中文符号啊
			
//先经过计算，100/0.3=333，然后通过实测，发现正常是在300左右
if(ErrorInt > 300) ErrorInt = 300;
if(ErrorInt < -300) ErrorInt = -300;	//这里我第一次写了ErrorInt=300，所以在负向调控时会出现剧烈抖动
			
//PID计算（调控力度）
Out = Kp*Error0+Ki*ErrorInt+Kd*(Error0-Error1);
//输出限幅
if(Out>100) Out=100;
if(Out<-100) Out=-100;
			
if(MotorFlag)
{
  //模拟电机正常工作
  Motor_SetPWM(Out);	//因为这个函数参数的有效范围是-100~100，所以输出限幅就   是-100~100.
}else
{
  //模拟电机断电或者故障
  Motor_SetPWM(0);
}
```

## 位置式PID定位置控制-积分分离

```c
//执行PID调控
Actual += Encoder_Get_Filtered();
//获取本次误差和上次误差
Error1 = Error0;
Error0 = Target-Actual;
//计算累计的误差积分(注意是小于）
if(fabs(Ki)>EPSILON && fabs(Error0)<50)
{
	ErrorInt+=Error0;
}else
{
	ErrorInt = 0;
}
//PID计算（调控力度）
Out = Kp*Error0+Ki*ErrorInt+Kd*(Error0-Error1);
//输出限幅
if(Out>100) Out=100;
if(Out<-100) Out=-100;
			
Motor_SetPWM(Out);	//因为这个函数参数的有效范围是-100~100，所以输出限幅就是-100~100.
```

## 位置式PID定位置控制-变速积分

```c
float C = 1 / (0.2 * fabs(Error0) + 1);
			
//计算累计的误差积分
ErrorInt+= C * Error0;
//PID计算（调控力度）
Out = Kp*Error0+Ki*ErrorInt+Kd*(Error0-Error1);
```

## 位置式PID定位置控制-微分先行

微分项输出现有一个正向的比较大的尖峰，然后再变成负向的，逐渐减少的曲线。每次目标值切换都必然产生一个数据点的尖峰，且目标值变化越大，变化越快，Kd越大，这个尖峰就越大。把Kp直接给0，现在是纯D项控制，可是D项仍然有输出，转盘也会超目标位置旋转，这个输出就全部是目标值改变导致的了，如果D项的作用是给实际值加阻尼，那么现在纯D项控制，单独改变目标值，D项应该是不会有输出的，这才符合阻尼的效果。

**微分先行要解决的问题**

普通PID的微分项对误差进行微分，当目标值大幅度跳变时，误差也会瞬间大幅度跳变，这回导致微分项突然输出一个很大的调控力，如果系统的目标值频繁大幅度切换，则此时的微分项不利于系统稳定。

**微分先行实现思路**

将对误差的微分替换为对实际值的微分。

```c
//DifOut = Kd*(Error0-Error1);
//微分先行，对实际值求微分
DifOut = -Kd*(Actual-Actual1);
			
//PID计算（调控力度）
Out = Kp*Error0+Ki*ErrorInt+DifOut;
```


## 位置式PID定位置控制-不完全微分

**不完全微分要解决的问题**

传感器获取的实际值经常会收到噪声干扰，而PID控制器中的微分项对噪声最为敏感，这些噪声干扰可能会导致微分项输出抖动，进而影响系统性能。

**不完全微分实现思路**

给微分项加入一阶惯性单元（低通滤波器）

```c
DifOut = Kd*(Error0-Error1);
//不完全微分
float a = 0.9;	//新的DifOut=0.1倍的本次D项+0.9倍的上次D项
DifOut = (1-a)*Kd*(Error0-Error1)+DifOut;
			
//PID计算（调控力度）
Out = Kp*Error0+Ki*ErrorInt+DifOut;
```

## 位置式PID定位置控制-输出偏移-输入死区

定位置PID控制，最终位置稳定下来会有一些误差无法消除，这个问题的其中一个解决办法就是加入I项，同时加入积分分离；但是用输出偏移也可以解决，而且位置稳定了Out也会有个很小的输出，意思是电机虽然停下来了，但它仍然收到一个持续的很小的驱动力，这个很小的驱动力其实时P项输出的。P项检测到有误差所以输出一个和误差大小成比例的值，但是误差比较小，这个输出值也比较小，最终作用到电机上，由于电机有摩擦力，电机启动需要一定的力度，所以这个很小的力带不动电机，所以实际位置和目标位置无法完全重合，还有产生不必要的功耗，为了避免能量浪费，我们希望电机不动时，Out值应该要变为0。

不过加入偏移，位置停下来时，还会频繁的调控，造成抖动。

现在使用了输出偏移和输入死区，首先是位置停下来的误差，目前的误差取决于输入死区大小，死区阈值给5，那么最终停下来之后，误差就不可能超过5。输出偏移和输入死区实现的效果是，在不加入I项的同时，还可以达到一个可控的误差区间，这样做的好处是避免了I项的滞后性。然后位置会停下来，Out必定是0，因为我们加入了输出偏移的逻辑，Out只要不为0，位置必然会动，那位置停下来，Out必然是0了，这样就解决了位置停下来，Out还不为0的问题。

**输出偏移要解决的问题**

对于一些启动需要一定力度的执行器，若输出值比较小，执行器可能完全没动作，这可能会引起调控误差，同时会降低系统相应速度。

**输出偏移实现思路**

若输出值为0，则正常输出0，不进行调控；若输出值非0，则给输出值加一个固定偏移，跳过执行器无动作的阶段。

**输入死区要解决的问题**

在某些系统中，输入的目标值或实际值有微小的噪声波动，或者系统有一定的滞后，这些情况可能会导致执行器在误差很小时频繁调控，不能稳定下来。

**输入死区实现思路**

若误差绝对值小于一个限度，则固定输出0.不进行调控，也就是容忍一定的调控误差。

```c
if(fabs(Error0)<5)
{
	Out = 0;
}else
{
	//计算累计的误差积分
	if(fabs(Ki)>EPSILON)
	{
		ErrorInt+=Error0;
	}else
	{
		ErrorInt = 0;
	}
	//PID计算（调控力度）
	Out = Kp*Error0+Ki*ErrorInt+Kd*(Error0-Error1);
				
	//输出偏移
	if(Out>0)
	{
		Out += 6;
	}else if(Out<0)
	{
		Out -= 5;
	}else
	{
		Out = 0;
	}
}
```

## 双环PID定速定位置控制

同时调节外环和内环的PID参数是不可行的，所以双环PID的调参要按照顺序来调，因为内环可以独立工作，所以我们要先调节内环参数，内环调节好了，我们再在内环的基础上，调节外环参数。在调节内环的时候，我们要把外环PID的代码注释掉，不要让外环工作。在内环定速控制的基础上，加入外环定位置控制的代码，同时开始调节外环的PID参数。加入外环，三个参数都给0，即使位置环什么都不干，用手拧转盘，已经感受到很强的阻力了。因为内环的速度环已经在维持目标速度为0的状态了，这就是双环PID的优势。

同样是PD控制器，单环PID位置会有调控误差，但是双环没有，是因为，双环的位置环的输出值作用于速度环的目标速度，因此无论位置环输出多小的值，电机都不可能不转。把速度环和电机看作一个整体，那这个整体的电机是不存在启动力度的，这样就不会像单环PD定位置控制那样，有最后一点误差消除不了，因此，双环PID有更高的准确性。用手强行去改变转盘，有很大的阻力，位置的调节是非常迅速和有力的，这是因为外力会同时改变转盘的速度和位置，内环和外环都会对外力做出相应，这比单环PID相应的更加迅速，相应力度也会更大，因此，双环PID具有更高的稳定性和响应速度。

```c
void TIM1_UP_IRQHandler(void)
{
	static uint16_t Count1,Count2;
	if (TIM_GetITStatus(TIM1, TIM_IT_Update) == SET)
	{
		//每隔1ms调用一次Key_Tick()，用于Key模块的内部检测
		Key_Tick();
		
		//内环
		Count1++;
		if(Count1>=10)	//40:20-32,20:20-16
		{				//每隔40ms读取一次编码器，值为32，意思是速度为32个边沿/40ms
			Count1=0;	//每隔20ms读取一次编码器，值为16，意思是速度为16个边沿/20ms，这两个表示的速度是一样的
			
			//执行PID调控
			Speed = Encoder_Get_Filtered();
			Location += Speed;
			InnerActual = Speed;
			//获取本次误差和上次误差
			InnerError1 = InnerError0;
			InnerError0 = InnerTarget-InnerActual;
			//计算累计的误差积分
			if(fabs(InnerKi)>EPSILON)
			{
				InnerErrorInt+=InnerError0;
			}else
			{
				InnerErrorInt = 0;
			}
			//PID计算（调控力度）
			InnerOut = InnerKp*InnerError0+InnerKi*InnerErrorInt+InnerKd*(InnerError0-InnerError1);
			//输出限幅
			if(InnerOut>100) InnerOut=100;
			if(InnerOut<-100) InnerOut=-100;
			
			Motor_SetPWM(InnerOut);	//因为这个函数参数的有效范围是-100~100，所以输出限幅就是-100~100.
		}
			
		//外环
		Count2++;
		if(Count2>=10)	//40:20-32,20:20-16
		{				//每隔40ms读取一次编码器，值为32，意思是速度为32个边沿/40ms
			Count2=0;	//每隔20ms读取一次编码器，值为16，意思是速度为16个边沿/20ms，这两个表示的速度是一样的
			
			//执行PID调控
			OuterActual = Location;	
			//获取本次误差和上次误差
			OuterError1 = OuterError0;
			OuterError0 = OuterTarget-OuterActual;
			//计算累计的误差积分
			if(fabs(OuterKi)>EPSILON)	//在外环测试时，这里解除了注释，但是我一开始这里的Inner没有改成Outer，所以我用手转一下转盘，转盘就疯狂抖动旋转
				{							//是因为如果没有改，意味着判断外环积分要不要累加的条件竟然是内环Ki等不等于0，速度不等于0，所以Ki就不等于0，所以外环积分项就无限累加
				OuterErrorInt+=OuterError0;
			}else
			{
				OuterErrorInt = 0;
			}
			//PID计算（调控力度）
			OuterOut = OuterKp*OuterError0+OuterKi*OuterErrorInt+OuterKd*(OuterError0-OuterError1);
			//输出限幅,OuterOut的输出限幅应该要限制在InnerTarget的输入范围，如果想要以指定速度，旋转到指定位置停下来，可以从这里外环输出值的限幅来设定
			if(OuterOut>45) OuterOut=45;		//从45到20，发现速度就变慢了
			if(OuterOut<-45) OuterOut=-45;
			
			//Motor_SetPWM(OuterOut);
			//外环的输出值要作用于内环目标值
			InnerTarget = OuterOut;
		}
		TIM_ClearITPendingBit(TIM1, TIM_IT_Update);
	}
}
```

外环不直接控制电机。外环负责告诉内环：“我希望电机以多快的速度运动”，内环负责：“好的，我控制电机，让速度达到这个目标”。

外环限幅实际上是在限制：最大运动速度。

##双环PID调试问题

**问题**

外环测试时电机突然高速旋转。

**原因**

外环积分使能判断错误，误用了内环Ki。

**修改**

将InnerKi改为OuterKi。

**结果**

外环积分正常工作。

**以指定速度，旋转到指定位置，停下来，并且速度和位置都要能适应负载变化和外界干扰，速度应该怎么设定**

改变外环输出值的限幅。

## PID代码的封装

```c
PID_t Inner = {
	.Kp = 1.1,
	.Ki = 1.1,
	.Kd = 0,
	.OutMax = 100,
	.OutMin = -100,
};

PID_t Outer = {
	.Kp = 0.1,
	.Ki = 0,
	.Kd = 0.35,
	.OutMax = 45,
	.OutMin = -45,
};

void PID_Update(PID_t *p)
{
	p->Error1 = p->Error0;
	p->Error0 = p->Target - p->Actual;
	
	if(p->Ki != 0)
	{
		p->ErrorInt += p->Error0;
	}else
	{
		p->ErrorInt = 0;
	}
	
	p->Out = p->Kp * p->Error0 
			+ p->Ki * p->ErrorInt
			+ p->Kd * (p->Error0 - p->Error1);
	
	if(p->Out > p->OutMax) p->Out = p->OutMax;
	if(p->Out < p->OutMin) p->Out = p->OutMin;
}

Speed = Encoder_Get_Filtered();
Location += Speed;
Inner.Actual = Speed;
			
PID_Update(&Inner);
			
Motor_SetPWM(Inner.Out);

Outer.Actual = Location;	
			
PID_Update(&Outer);
			
//Motor_SetPWM(Outer.Out);
//外环的输出值要作用于内环目标值
Inner.Target = Outer.Out;
```

**结构体变量或结构体指针**

.用于结构体变量；->用于结构体指针。

本质区别：

- .：我手里只有这个结构体本身
- ->：我手里只有这个结构体的地址，通过地址找到结构体

## 基础驱动代码-角度传感器

读取角度传感器，使用ADC1，改成通道8，摆杆竖直时AD读值时2090左右。
读取电位器旋钮，使用ADC2。
拔掉J1，角度传感器，可以看到引脚悬空，AD值在2048附近，极易收到干扰，是不可靠的。

```c
void AD_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	
	RCC_ADCCLKConfig(RCC_PCLK2_Div6);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStructure);
	
	ADC_RegularChannelConfig(ADC1, ADC_Channel_8, 1, ADC_SampleTime_55Cycles5);
	
	ADC_InitTypeDef ADC_InitStructure;
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
	ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;
	ADC_InitStructure.ADC_ScanConvMode = DISABLE;
	ADC_InitStructure.ADC_NbrOfChannel = 1;
	ADC_Init(ADC1, &ADC_InitStructure);
	
	ADC_Cmd(ADC1, ENABLE);
	
	ADC_ResetCalibration(ADC1);
	while (ADC_GetResetCalibrationStatus(ADC1) == SET);
	ADC_StartCalibration(ADC1);
	while (ADC_GetCalibrationStatus(ADC1) == SET);
}

uint16_t AD_GetValue(void)
{
	ADC_SoftwareStartConvCmd(ADC1, ENABLE);
	while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);
	return ADC_GetConversionValue(ADC1);
}

void RP_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC2, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	RCC_ADCCLKConfig(RCC_PCLK2_Div6);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2 | GPIO_Pin_3 | GPIO_Pin_4 | GPIO_Pin_5;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
		
	ADC_InitTypeDef ADC_InitStructure;
	ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;
	ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;
	ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
	ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;
	ADC_InitStructure.ADC_ScanConvMode = DISABLE;
	ADC_InitStructure.ADC_NbrOfChannel = 1;
	ADC_Init(ADC2, &ADC_InitStructure);
	
	ADC_Cmd(ADC2, ENABLE);
	
	ADC_ResetCalibration(ADC2);
	while (ADC_GetResetCalibrationStatus(ADC2) == SET);
	ADC_StartCalibration(ADC2);
	while (ADC_GetCalibrationStatus(ADC2) == SET);
}

uint16_t RP_GetValue(uint8_t n)
{
	//通过参数n的不同来选择RP1,2,3,4分别对应的ADC通道(注：RP一般代表电位器)
	if(n==1)
	{
		ADC_RegularChannelConfig(ADC2, ADC_Channel_2, 1, ADC_SampleTime_55Cycles5);
	}
	else if(n==2)
	{
		ADC_RegularChannelConfig(ADC2, ADC_Channel_3, 1, ADC_SampleTime_55Cycles5);
	}
	else if(n==3)
	{
		ADC_RegularChannelConfig(ADC2, ADC_Channel_4, 1, ADC_SampleTime_55Cycles5);
	}
	else if(n==4)
	{
		ADC_RegularChannelConfig(ADC2, ADC_Channel_5, 1, ADC_SampleTime_55Cycles5);
	}
	ADC_SoftwareStartConvCmd(ADC2, ENABLE);
	while (ADC_GetFlagStatus(ADC2, ADC_FLAG_EOC) == RESET);
	return ADC_GetConversionValue(ADC2);
}
```
