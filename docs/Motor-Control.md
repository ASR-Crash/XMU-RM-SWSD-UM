<center><font size = 7>电机控制</font></center>

在机器人运动中，电机控制为最重要的部分之一，机器人的各种运动都离不开电机的运作。写程序的时候最忌讳重复造轮子，这里就以已有的步兵标准代码为例，利用已经实现的方法对电机进行一些控制。

我们常用的电机有三种M3508电机（需搭配C6020电调使用）、GM6020电机、M2006电机（需搭配C6010电调使用）

# 头文件

电机类头文件为 motor.h  ，里面构造了实现了电机类的一些实现

## 参数定义

```c++
constexpr auto MAXSPEED = 4000;
constexpr auto ADJUSTSPEED = 3000;

enum { ID1 = 0x201, ID2, ID3, ID4, ID5, ID6, ID7, ID8};
enum { pre = 0, now };
enum pid_mode{ speed = 0, position };
enum motor_type{ M3508, M3510, M2310, EC60, M6623, M6020 }; //M2310也是M2006
enum motor_mode { SPD, POS, ACE };

#define SQRTF(x) ((x)>0?sqrtf(x):-sqrtf(-x))			
#define T 1.e-3f
```

该部分为一些数值的预定义，包括一些ID的值枚举，ID代表着各个电机的标识符，具体数据见 [电机参数](../refs/电机参数.md)

## Motor类

```c++
class Motor
{
	/*具体实现*/
}
```

public部分中定义了电机的构造函数，电机控制方法、以及电机的三种操作模式

### 构造函数

```C++
//构造函数1：需填入五个参数; 电机运动过程由速度环与位置环共同调节
//一般在GM6020电机上使用此构造函数
Motor(const type_t type,const motor_mode mode,const uint32_t id, PID _speed, PID _position)
    : ID(id)
        , type(type)
        , mode(mode)
    {
        getmax(type);
        memcpy(&pid[speed], &_speed, sizeof(PID));
        memcpy(&pid[position], &_position, sizeof(PID));
    }
//构造函数2：需填入4个参数; 电机运动过程仅由速度环调节
//一般在M3508、M2006电机上使用此
Motor(const type_t type, const motor_mode mode, const uint32_t id, PID _speed)
    : ID(id)
        , type(type)
        , mode(mode)
    {
        getmax(type);
        memcpy(&pid[speed], &_speed, sizeof(PID));
    }
```

### 具体实现

```C++
/*
 *根据电机模型来实现其功能
 *para1: idata[][8]电调的接受报文
 *para2: odata 输出报文
 
 *实现功能:可以根据C620电调的收发报文控制电机的运转
*/
void Ontimer(uint8_t idata[][8], uint8_t* odata)
{
    const uint32_t ID = this->ID - ID1;
    angle[now] = getword(idata[ID][0], idata[ID][1]);

    if(type == EC60)curspeed = static_cast<float>(getdeltaa(angle[now] - angle[pre])) / T / 8192.f * 60.f;
    else curspeed = getword(idata[ID][2], idata[ID][3]);
    ///angle[pre]&angle[now]&curspeed are available.
    if(mode == ACE)//accurate mode
    {
        curcircle += getdeltaa(angle[now] - angle[pre]);
        if (spinning)
        {
            const int32_t diff = curcircle - setcircle;
            if (std::fabs(diff) >= 4096) setspeed = -adjspeed * SQRTF((float)diff / (float)spinning);
            else
            {
                // = std::fabs(diff);
                setangle = 8192 + ((diff < 0) ? -1 : 1)*(std::abs(diff) % 8192);
                spinning = 0;
            }
        }
        else
        {
            setcircle = curcircle;
            const float error = getdeltaa(setangle - angle[now]);
            if (std::fabs(error) < 30.f)setspeed = 0.0;
            else
            {
                setspeed = pid[position].Position(//pid[position].Filter(
                    getdeltaa(setangle - angle[now]));
                setspeed = setrange(setspeed, maxspeed);
            }
        }

        current += pid[speed].Delta(setspeed - curspeed);
        current = setrange(current, maxcurrent);
    }
    else if(mode == POS)//position mode
    {
        const float error = getdeltaa(setangle - angle[now]);
        //if (fabs(error) < 30.f)setspeed = 0.0;
        //else
        {
            setspeed = pid[position].Position((
                getdeltaa(setangle - angle[now])));
            setspeed = setrange(setspeed, maxspeed);
        }
        current = pid[speed].Position(setspeed - curspeed);
        current = setrange(current, maxcurrent);
    }
    else if(mode == SPD)//speed mode
        //proceed speed any way.
    {
        current += pid[speed].Delta(setspeed - curspeed);
        current = setrange(current, maxcurrent);
    }

    angle[pre] = angle[now];
    odata[ID * 2] = (current & 0xff00) >> 8;
    odata[ID * 2 + 1] = current & 0x00ff;
}
```

还有一些其他的功能函数


```c++
/*
 *用于根据电机型号返回上限值
*/
void getmax(const type_t type)
{
    adjspeed = 3000;
    switch(type)
    {
        case M3508:
        case M3510:
            maxcurrent = 13000;
            maxspeed = 9000;
            break;
        case M2310:
            maxcurrent = 13000;
            maxspeed = 9000;
            break;
        case EC60:
            maxcurrent = 5000;
            maxspeed = 300;
            break;
        case M6623:
            maxcurrent = 5000;
            maxspeed = 300;
            break;
        case M6020:
            maxcurrent = 30000;
            maxspeed = 80;
            adjspeed = 80;
            break;
        default:;
    }
}
```

