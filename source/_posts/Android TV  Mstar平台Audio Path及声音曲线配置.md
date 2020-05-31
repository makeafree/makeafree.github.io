## [Android TV : Mstar平台Audio Path及声音曲线配置](https://www.cnblogs.com/blogs-of-lxl/p/12978759.html)

## 一、Audio Path配置

　　在HW 修改电路之后，SW 通常也需要重新修改匹配硬件，这样才能保证功能的正常使用。

　　下面以VGA 通过的line-lin 为例：　　

　　![img](https://img2020.cnblogs.com/blog/821933/202005/821933-20200528095018201-1130768780.png)

　　从原理图上看，VGA通道的PC audio in 是属于芯片端的pin 脚是Y3，AA4，对应的 port 是line-in 第 0 路。

　　mstar 平台 audio 的映射关系包含audio in，audio out，以及内部 audio mux。对应的是board.h 文件中的三个结构体：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// audio-in , 右侧对应的是通道，左侧是对应的声音进入的port 
static AudioMux_t BOARD_AudioMux_t[BOARD_AUDIO_INPUT_SOURCE_TYPE_SIZE] =
{
    {AUDIO_DSP1_DVB_INPUT},    //AUDIO_SOURCE_DTV
    {AUDIO_DSP1_DVB_INPUT},    //AUDIO_SOURCE_DTV2
    {AUDIO_DSP4_SIF_INPUT},    //AUDIO_SOURCE_ATV
    {AUDIO_AUIN4_INPUT},       //AUDIO_SOURCE_PC
    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_YPbPr
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_YPbPr2
    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_AV
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_AV2
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_AV3
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_SV
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_SV2
    {AUDIO_AUIN4_INPUT},       //AUDIO_SOURCE_SCART
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_SCART2
    {AUDIO_HDMI_INPUT},        //AUDIO_SOURCE_HDMI
    {AUDIO_HDMI_INPUT},        //AUDIO_SOURCE_HDMI2
    {AUDIO_HDMI_INPUT},        //AUDIO_SOURCE_HDMI3
    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_DVI
    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_DVI2
    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_DVI3
    {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_KTV
};
//  audio mux，右侧对应的通道也即input source，左侧对应的是耳机/喇叭的port 
static AudioMux_t BOARD_AudioPath_t[BOARD_AUDIO_PATH_TYPE_SIZE] =
{
    {AUDIO_T3_PATH_I2S},       //AUDIO_PATH_MAIN_SPEAKER
    {AUDIO_T3_PATH_AUOUT1},    //AUDIO_PATH_HP
    {AUDIO_T3_PATH_AUOUT0},    //AUDIO_PATH_LINEOUT
    {AUDIO_PATH_NULL},         //AUDIO_PATH_SIFOUT
    {AUDIO_PATH_NULL},         //AUDIO_PATH_SCART1 = SIF out
    {AUDIO_T3_PATH_AUOUT0},    //AUDIO_PATH_SCART2 = Lineout
    {AUDIO_T3_PATH_SPDIF},     //AUDIO_PATH_SPDIF
    {AUDIO_PATH_NULL},         //AUDIO_PATH_HDMI
    {AUDIO_T3_PATH_MIXER_MAIN},        // AUDIO_PATH_MIXER_MAIN
    {AUDIO_T3_PATH_MIXER_SECONDARY},        // AUDIO_PATH_MIXER_SECONDARY
    {AUDIO_PATH_NULL},                  // AUDIO_PATH_7
    {AUDIO_T3_PATH_MIXER_DMA_IN},        // AUDIO_PATH_MIXER_DMA_IN
};
// audio-out， 是输出path 的选择
static AudioOutputType_t BOARD_AudioOutputType_t[BOARD_AUDIO_OUTPUT_TYPE_SIZE] =
{
    {AUDIO_I2S_OUTPUT},    //AUDIO_PATH_MAIN_SPEAKER
    {AUDIO_AUOUT1_OUTPUT},    //AUDIO_PATH_HP
    {AUDIO_AUOUT0_OUTPUT},    //AUDIO_PATH_LINEOUT
    {AUDIO_NULL_OUTPUT},    //AUDIO_PATH_SIFOUT
    {AUDIO_NULL_OUTPUT},    //AUDIO_PATH_SCART1 = SIF out
    {AUDIO_AUOUT0_OUTPUT},    //AUDIO_PATH_SCART2 = Lineout
};
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　从上面的信息可以看到，line-in 里面AUDIO_AUIN0_INPUT port 对应了YPBPR，AV，DVI，DVI2，DVI3 几个通道，未看到VGA通道，配置audio in， out ，mux 信息之后，需要在系统中生效就必须初始化，设置到寄存器中去，继续查到找到三个结构体，分别保存在systeminfo模块的m_pAudioMuxInfo，m_pAudioPathInfo，m_pAudioOutputTypeInfo三个成员中，通过下面接口获取：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
@vendor\mstar\supernova\projects\systeminfo\src\SystemInfo.cpp
const AudioMux_t* SystemInfo::GetAudioInputMuxInfo()
{
    if(m_pAudioMuxInfo != NULL)
    {
        return m_pAudioMuxInfo;
    }

    ASSERT(0);
}

const AudioPath_t* SystemInfo::GetAudioPathInfo()
{
    if(m_pAudioPathInfo != NULL)
    {
        return m_pAudioPathInfo;
    }

    ASSERT(0);
}

const AudioOutputType_t* SystemInfo::GetAudioOutputTypeInfo()
{
    if(m_pAudioOutputTypeInfo != NULL)
    {
        return m_pAudioOutputTypeInfo;
    }

    ASSERT(0);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　接着在_InputSourceTypeToAudioInputType（vendor\mstar\supernova\projects\customization\MStarSDK\audio\mapi_audio_customer.cpp）接口中获取，根据通道进行设置：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
static AUDIO_INPUT_TYPE _InputSourceTypeToAudioInputType(MAPI_INPUT_SOURCE_TYPE eSourceType)
{
    const AudioMux_t* const p_AudioInputMux = mapi_syscfg_fetch::GetInstance()->GetAudioInputMuxInfo();
    AUDIO_INPUT_TYPE eAudioInputType = AUDIO_NULL_INPUT;
    MAPI_U32 u32Port = 0xFF;

    switch(eSourceType)
    {
        case MAPI_INPUT_SOURCE_DTV:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_DTV].u32Port;
            break;
        }
        ........
        case MAPI_INPUT_SOURCE_YPBPR:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_YPBPR].u32Port;
            break;
        }
        case MAPI_INPUT_SOURCE_YPBPR2:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_YPBPR2].u32Port;
            break;
        }

        case MAPI_INPUT_SOURCE_VGA:
        case MAPI_INPUT_SOURCE_VGA2:
        case MAPI_INPUT_SOURCE_VGA3:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_PC].u32Port;
            break;
        }
        case MAPI_INPUT_SOURCE_HDMI:
        case MAPI_INPUT_SOURCE_HDMI2:
        case MAPI_INPUT_SOURCE_HDMI3:
        case MAPI_INPUT_SOURCE_HDMI4:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_HDMI].u32Port;
            break;
        }
        case MAPI_INPUT_SOURCE_DVI:
        {
            u32Port = p_AudioInputMux[MAPI_AUDIO_SOURCE_DVI].u32Port;
            break;
        }

        ......
    }

    eAudioInputType = _u32PortToAudioInputType(u32Port);
    return eAudioInputType;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　到这里可以发现VGA，VGA2，VGA3 使用的是 audio in 结构体中的source （MAPI_AUDIO_SOURCE_PC） 对应的的port ：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
diff --git a/serials/MST160D_10ABQM_18443_DTMB_AH/Board.h b/serials/MST160D_10ABQM_18443_DTMB_AH/Board.h
index 24b27d5..da095f1 100755
--- a/serials/MST160D_10ABQM_18443_DTMB_AH/Board.h
+++ b/serials/MST160D_10ABQM_18443_DTMB_AH/Board.h
@@ -820,7 +820,7 @@ static AudioMux_t BOARD_AudioMux_t[BOARD_AUDIO_INPUT_SOURCE_TYPE_SIZE] =
     {AUDIO_DSP1_DVB_INPUT},    //AUDIO_SOURCE_DTV
     {AUDIO_DSP1_DVB_INPUT},    //AUDIO_SOURCE_DTV2
     {AUDIO_DSP4_SIF_INPUT},    //AUDIO_SOURCE_ATV
-    {AUDIO_AUIN4_INPUT},       //AUDIO_SOURCE_PC
+    {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_PC
     {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_YPbPr
     {AUDIO_NULL_INPUT},        //AUDIO_SOURCE_YPbPr2
     {AUDIO_AUIN0_INPUT},       //AUDIO_SOURCE_AV
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　因此将AUDIO_AUIN4_INPUT 换成AUDIO_AUIN0_INPUT 即可。

　　如果 speaker 和 headphone 同一路硬件输出，优先以speaker 为主，同时AUDIO_OUTPUT_HP 的audio path 可以配置为NULL。　

　　![img](https://img2020.cnblogs.com/blog/821933/202005/821933-20200528101826269-580895933.png)

　　更多mstar 平台音频属性，音效设置详参文件：drvAUDIO_if.h，mapi_audio_datatype.h，apiAUDIO.h，mapi_audio.h，mapi_audio_customer.cpp，mapi_audio.cpp,MSrv_SSSound.cpp

 

## 二、声音曲线配置

　　mstar平台在配置完音频通道，调通功放后，需要调整声音曲线和声音功率。声音功率是调整音量逻辑最大值(100)时，喇叭(负载)功率的最大值，而声音曲线是音量0 ~ 100 整个区间，对应的音频输出的增益大小，所以声音功率调试也是曲线调试的一部分。



**（1）增益寄存器**

　　 例如，我当前平台音频输出对应的bank 是0x112D

　　![img](https://img2020.cnblogs.com/blog/821933/202005/821933-20200528110242767-479386450.png)

 

　　其中，音频输出通道有6个，如下

 　![img](https://img2020.cnblogs.com/blog/821933/202005/821933-20200528110319294-432976471.png)

 　两个寄存器组成16 位，来完成增益值的配置。

 

**（2）增益调整**
　　从上面的excel表格中，可以知悉每个音频通道的增益调整的寄存器的16个bit位。接好喇叭(负载的阻值要和客户要求一样，例如；8欧姆)，播放1KHZ -12 db 的音源，接上示波器量负载电压。

　　最大功率调整：将电视音量调整到最大值100，再调整对应输出通道寄存器值，观察电压变化，并计算功率，达到客户要求的功率为止。

　　曲线调整：最大功率就是音量为100时，寄存器的增益值。接着在调整0~99区间对应的增益值。

**（3）音频曲线配置**

　　在customer_1.ini 中配置

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
TypeID = 5;
[VolumeCurve]
bEnabled = TRUE;
u8Volume_Int[101] = \
    { \
        INT_LINE1, \
        INT_LINE2, \
        INT_LINE3, \
        INT_LINE4, \
        INT_LINE5, \
        INT_LINE6, \
        INT_LINE7, \
        INT_LINE8, \
        INT_LINE9, \
        INT_LINEa, \
        INT_LINEb  \
    };

 u8Volume_Fra[101] = \
    {  \
        FRA_LINE1, \
        FRA_LINE2, \
        FRA_LINE3, \
        FRA_LINE4, \
        FRA_LINE5, \
        FRA_LINE6, \
        FRA_LINE7, \
        FRA_LINE8, \
        FRA_LINE9, \
        FRA_LINEa, \
        FRA_LINEb  \
    };
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　数组中的宏可以根据不同功放进行客制化参数配置，如：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
INT_LINE1=0x7F
INT_LINE2='0x46,0x42,0x40,0x3E,0x3C,0x3a,0x38,0x36,0x34,0x32'
INT_LINE3='0x30,0x2E,0x2D,0x2C,0x2B,0x2A,0x29,0x28,0x27,0x26'
INT_LINE4='0x25,0x24,0x23,0x22,0x21,0x20,0x1F,0x1E,0x1E,0x1D'
INT_LINE5='0x1D,0x1C,0x1C,0x1B,0x1B,0x1A,0x1A,0x19,0x19,0x18'
INT_LINE6='0x18,0x17,0x17,0x16,0x16,0x15,0x15,0x15,0x14,0x14'
INT_LINE7='0x14,0x14,0x13,0x13,0x13,0x13,0x12,0x12,0x12,0x12'
INT_LINE8='0x11,0x11,0x11,0x11,0x10,0x10,0x10,0x10,0x0F,0x0F'
INT_LINE9='0x0F,0x0F,0x0F,0x0F,0x0F,0x0F,0x0E,0x0E,0x0E,0x0E'
INT_LINEa='0x0E,0x0E,0x0E,0x0E,0x0D,0x0D,0x0D,0x0D,0x0D,0x0D'
INT_LINEb='0x0D,0x0D,0x0C,0x0C,0x0C,0x0C,0x0C,0x0C,0x0C,0x0C'

FRA_LINE1=0x00
FRA_LINE2='0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00'
FRA_LINE3='0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00'
FRA_LINE4='0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x04,0x00,0x04'
FRA_LINE5='0x00,0x04,0x00,0x04,0x00,0x04,0x00,0x04,0x00,0x04'
FRA_LINE6='0x00,0x04,0x00,0x04,0x00,0x04,0x02,0x00,0x06,0x04'
FRA_LINE7='0x02,0x00,0x06,0x04,0x02,0x00,0x06,0x04,0x02,0x00'
FRA_LINE8='0x06,0x04,0x02,0x00,0x06,0x04,0x02,0x00,0x07,0x06'
FRA_LINE9='0x05,0x04,0x03,0x02,0x01,0x00,0x07,0x06,0x05,0x04'
FRA_LINEa='0x03,0x02,0x01,0x00,0x07,0x06,0x05,0x04,0x03,0x02'
FRA_LINEb='0x01,0x00,0x07,0x06,0x05,0x04,0x03,0x02,0x01,0x00'
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　配置文件在

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
mapi_audio_customer.cpp
void mapi_audio_customer::SetAbsoluteVolume(const MAPI_U8 u8Path, MAPI_U8 uPercent, const MAPI_U8 uReserve) const
{
    MAPI_U8  TransVolum_int = 0;
    MAPI_U8  TransVolum_fra = 0;

    UNUSED(uReserve);

    if(uPercent >= MAPI_AUDIO_VOLUME_ARRAY_NUMBER)
    {
        uPercent = MAPI_AUDIO_VOLUME_ARRAY_NUMBER - 1;
        printf("SetAbsoluteVolume: uPercent value overflow!!!\n");
    }

    //check if system configuration provide customized volume curve.
    const VolumeCurve_t* const curve = mapi_syscfg_fetch::GetInstance()->GetVolumeCurve();
    if ((curve != NULL) && (curve->bEnabled == 1))
    {
        TransVolum_int = curve->u8Volume_Int[uPercent];
        TransVolum_fra = curve->u8Volume_Fra[uPercent];
    }
    else // Use default volume table.
    {
        TransVolum_int = (MAPI_U8)(u8Volume[uPercent] >> 8);
        TransVolum_fra = (MAPI_U8)(u8Volume[uPercent] & 0x00FF);
    }

    printf("\nSetAbsoluteVolume: u8Path = 0x%x\n",u8Path);
    printf("\nSetAbsoluteVolume: Set Volume int value = 0x%x\n", TransVolum_int);
    printf("\nSetAbsoluteVolume: Set Volume fra value = 0x%x\n", TransVolum_fra);
    MApi_AUDIO_SetAbsoluteVolume(u8Path, TransVolum_int, TransVolum_fra);
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　通过mapi_syscfg_fetch::GetInstance()->GetVolumeCurve()即获取ini配置声音曲线表，u8Patch 对应的就是 lineout0，lineout1，lingout2，lineout3…

　　最后通过MApi_AUDIO_SetAbsoluteVolume(u8Path, TransVolum_int, TransVolum_fra) 设置到主芯片寄存器：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
////////////////////////////////////////////////////////////////////////////////
/// @ingroup Audio_BASIC_SOUNDEFFECT
/// @brief \b Function \b Name: MDrv_SOUND_AbsoluteVolume()
/// @brief \b Function \b Description: This routine is used to set the absolute u8Volume of audio u8Path.
/// @param u8Path      \b : for audio u8Path0 ~ u8Path6
/// @param u8Vol1      \b : MSB 7-bit register value of 10-bit u8Volume
///                         range from 0x00 to 0x7E , gain: +12db to   -114db (-1 db per step)
/// @param u8Vol2      \b : LSB 3-bit register value of 10-bit u8Volume
///                         range from 0x00 to 0x07 , gain:  -0db to -0.875db (-0.125 db per step)
////////////////////////////////////////////////////////////////////////////////
 void    MApi_AUDIO_SetAbsoluteVolume( MS_U8 u8Path, MS_U8 u8Vol1, MS_U8 u8Vol2 );
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

　　当使用外挂混音，需要固定主芯片输出时，以下通过MApi_XC_WriteByte 直接固定了lineout0(0x112D) 这一路的输出，

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
MAPI_U8 mapi_audio_customer::fixOutputVolumeOffset() const
{
    #define REGADDR (0x112D00)
    
    MAPI_U8  TransVolum_int       = 0;
    MAPI_U8  TransVolum_fra       = 0;
    MAPI_U16  u16Temp             = 0x0000;
    const MAPI_U16  u16Vga        = 0x0c00;
    const MAPI_U16  u16Hdmi       = 0x0400;
    const MAPI_U16  u16tv         = 0x04c0; // atv, dtv
    const MAPI_U16  u16Storage    = 0x0b80;
    const MAPI_U16  u16Cvbs       = 0x0a80;

    if(MAPI_INPUT_SOURCE_VGA == m_mainCurInputSrc)
    {
        u16Temp = u16Vga;
    }
    else if(MAPI_INPUT_SOURCE_ATV == m_mainCurInputSrc || MAPI_INPUT_SOURCE_DTV == m_mainCurInputSrc)
    {
        u16Temp = u16tv;
    }
    else if(MAPI_INPUT_SOURCE_CVBS <= m_mainCurInputSrc && MAPI_INPUT_SOURCE_CVBS_MAX > m_mainCurInputSrc)
    {
        u16Temp = u16Cvbs;
    }
    else if (MAPI_INPUT_SOURCE_YPBPR <= m_mainCurInputSrc && MAPI_INPUT_SOURCE_YPBPR_MAX > m_mainCurInputSrc)
    {
        u16Temp = u16Cvbs;
    }
    else if(MAPI_INPUT_SOURCE_HDMI <= m_mainCurInputSrc && MAPI_INPUT_SOURCE_HDMI_MAX > m_mainCurInputSrc)
    {
        u16Temp = u16Hdmi;
    }
    else if(MAPI_INPUT_SOURCE_STORAGE == m_mainCurInputSrc)
    {
        u16Temp = u16Storage;
    }
    else
    {
        return 0;
    }

    TransVolum_int = (u16Temp >> 8);
    TransVolum_fra = (u16Temp & 0x00FF);

    printf("[%s][%d] fix the android output to 4.6db \n",__FUNCTION__,__LINE__);
    printf("[%s][%d] source id: %d, reg[0x112D].value = 0x%04x \n",__FUNCTION__,__LINE__,m_mainCurInputSrc,u16Temp);
    printf("[%s][%d] int: 0x%02x, fra: 0x%02x \n",__FUNCTION__,__LINE__,TransVolum_int,TransVolum_fra);

    MApi_XC_WriteByte(REGADDR , TransVolum_fra);
    MApi_XC_WriteByte(REGADDR + 1, TransVolum_int);
    return 1;
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)