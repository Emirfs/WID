---
name: new-project
description: Yeni bir STM32 gömülü sistem projesi kurar. Proje klasöründe CLAUDE.md, PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md, CubeIDE kaynak dosyaları ve .ioc dosyası oluşturur.
---

# /new-project — Yeni STM32 Projesi Kurulumu

## Adım 1: Kullanıcıdan Bilgi Topla

Şu soruları **tek tek** sor (cevap alınmadan bir sonrakine geçme):

1. "Proje adı nedir?" (klasör adı olarak kullanılacak)
2. "Hangi STM32 MCU kullanıyorsunuz?" (örn: STM32F407VGTx, STM32H743ZI, STM32F446RE)
3. "MCU paketi nedir?" (örn: LQFP100, LQFP64, TFBGA216) — aşağıdaki tablodan biliniyorsa atla
4. "Projenin amacı nedir? (kısa açıklama)"
5. "Sistem clock frekansı nedir? (örn: 25MHz, 168MHz, 180MHz)"
6. "FreeRTOS kullanılacak mı?"
7. "Projenin ana gereksinimleri neler?" (virgülle ayırarak liste ver)

---

## Adım 2: MCU Parametrelerini Belirle

Kullanıcının MCU modeline göre aşağıdaki tablodan değerleri seç:

| MCU Modeli      | Mcu.CPN          | Mcu.Name                 | Mcu.Package | Mcu.Family | FirmwarePackage                  | Flash   | RAM    | Max Clock |
|-----------------|------------------|--------------------------|-------------|------------|----------------------------------|---------|--------|-----------|
| STM32F407VGTx   | STM32F407VGT6    | STM32F407V(E-G)Tx        | LQFP100     | STM32F4    | STM32Cube FW_F4 V1.28.3          | 1024K   | 128K   | 168 MHz   |
| STM32F407ZGTx   | STM32F407ZGT6    | STM32F407Z(E-G)Tx        | LQFP144     | STM32F4    | STM32Cube FW_F4 V1.28.3          | 1024K   | 128K   | 168 MHz   |
| STM32F446RETx   | STM32F446RET6    | STM32F446RETx            | LQFP64      | STM32F4    | STM32Cube FW_F4 V1.28.3          | 512K    | 128K   | 180 MHz   |
| STM32F103C8Tx   | STM32F103C8T6    | STM32F103C8Tx            | LQFP48      | STM32F1    | STM32Cube FW_F1 V1.8.6           | 64K     | 20K    | 72 MHz    |
| STM32F103RBTx   | STM32F103RBT6    | STM32F103RBTx            | LQFP64      | STM32F1    | STM32Cube FW_F1 V1.8.6           | 128K    | 20K    | 72 MHz    |
| STM32H743ZITx   | STM32H743ZIT6    | STM32H743ZITx            | LQFP144     | STM32H7    | STM32Cube FW_H7 V1.11.2          | 2048K   | 1024K  | 480 MHz   |
| STM32L432KCUx   | STM32L432KCU6    | STM32L432KCUx            | UFQFPN32    | STM32L4    | STM32Cube FW_L4 V1.18.1          | 256K    | 64K    | 80 MHz    |
| STM32G474RETx   | STM32G474RET6    | STM32G474RETx            | LQFP64      | STM32G4    | STM32Cube FW_G4 V1.5.2           | 512K    | 128K   | 170 MHz   |

Tabloda yoksa: CPN = modeli sonundaki harfi rakama çevir (T→6), Name = model adını koru.

---

## Adım 3: CubeIDE Proje Dosyaları Oluştur

Klasör yapısı:

```
<proje_adı>/
├── .project                     ← Eclipse proje tanımlayıcısı (CubeIDE tanır)
├── <proje_adı>.ioc              ← CubeMX/CubeIDE konfigürasyon dosyası
├── Core/
│   ├── Src/
│   │   ├── main.c
│   │   ├── stm32<aile>xx_it.c
│   │   ├── stm32<aile>xx_hal_msp.c
│   │   ├── syscalls.c
│   │   └── sysmem.c
│   └── Inc/
│       ├── main.h
│       └── stm32<aile>xx_it.h
├── CLAUDE.md
├── PROJECT.md
├── STATE.md
├── REQUIREMENTS.md
├── ROADMAP.md
└── project_memory/
    ├── hardware.md
    ├── decisions.md
    ├── bugs.md
    ├── progress.md
    └── changelog.md
```

> **CubeIDE Notu:** "Generate Code" çalıştırıldığında şunlar otomatik eklenir:
> `Drivers/`, `startup_stm32<...>.s`, `<MCU>_FLASH.ld`, `Core/Src/system_stm32<aile>xx.c`, `Core/Inc/stm32<aile>xx_hal_conf.h`, `.cproject`

---

### .project

```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name><proje_adı></name>
	<comment></comment>
	<projects>
	</projects>
	<buildSpec>
		<buildCommand>
			<name>org.eclipse.cdt.managedbuilder.core.genmakebuilder</name>
			<triggers>clean,full,incremental,</triggers>
			<arguments>
			</arguments>
		</buildCommand>
		<buildCommand>
			<name>org.eclipse.cdt.managedbuilder.core.ScannerConfigBuilder</name>
			<triggers>full,incremental,</triggers>
			<arguments>
			</arguments>
		</buildCommand>
	</buildSpec>
	<natures>
		<nature>com.st.stm32cube.ide.mcu.MCUProjectNature</nature>
		<nature>com.st.stm32cube.ide.mcu.MCUCubeProjectNature</nature>
		<nature>org.eclipse.cdt.core.cnature</nature>
		<nature>com.st.stm32cube.ide.mcu.MCUCubeIdeServicesRevAev2ProjectNature</nature>
		<nature>com.st.stm32cube.ide.mcu.MCUAdvancedStructureProjectNature</nature>
		<nature>com.st.stm32cube.ide.mcu.MCUSingleCpuProjectNature</nature>
		<nature>com.st.stm32cube.ide.mcu.MCURootProjectNature</nature>
		<nature>org.eclipse.cdt.managedbuilder.core.managedBuildNature</nature>
		<nature>org.eclipse.cdt.managedbuilder.core.ScannerConfigNature</nature>
	</natures>
</projectDescription>
```

---

### <proje_adı>.ioc

Aşağıdaki tam şablonu kullan. `<...>` yer tutucularını Adım 2'deki tabloya göre doldur.

**STM32F4 / STM32F1 / STM32L4 / STM32G4 için:**

```
#MicroXplorer Configuration settings - do not modify
CAD.formats=
CAD.pinconfig=
CAD.provider=
File.Version=6
KeepUserPlacement=false
Mcu.CPN=<MCU_CPN>
Mcu.Family=<MCU_FAMILY>
Mcu.IP0=NVIC
Mcu.IP1=RCC
Mcu.IP2=SYS
Mcu.IPNb=3
Mcu.Name=<MCU_NAME>
Mcu.Package=<MCU_PACKAGE>
Mcu.Pin0=VP_SYS_VS_Systick
Mcu.PinsNb=1
Mcu.ThirdPartyNb=0
Mcu.UserConstants=
Mcu.UserName=<MCU_USERNAME>
MxCube.Version=6.15.0
MxDb.Version=DB.6.0.150
NVIC.BusFault_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.DebugMonitor_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.ForceEnableDMAVector=true
NVIC.HardFault_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.MemoryManagement_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.NonMaskableInt_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.PendSV_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.PriorityGroup=NVIC_PRIORITYGROUP_0
NVIC.SVCall_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
NVIC.SysTick_IRQn=true\:0\:0\:false\:false\:true\:true\:true\:false
NVIC.UsageFault_IRQn=true\:0\:0\:false\:false\:true\:true\:false\:false
PinOutPanel.RotationAngle=0
ProjectManager.AskForMigrate=true
ProjectManager.BackupPrevious=false
ProjectManager.CompilerLinker=GCC
ProjectManager.CompilerOptimize=6
ProjectManager.ComputerToolchain=false
ProjectManager.CoupleFile=false
ProjectManager.CustomerFirmwarePackage=
ProjectManager.DefaultFWLocation=true
ProjectManager.DeletePrevious=true
ProjectManager.DeviceId=<MCU_USERNAME>
ProjectManager.FirmwarePackage=<FIRMWARE_PACKAGE>
ProjectManager.FreePins=false
ProjectManager.HalAssertFull=false
ProjectManager.HeapSize=0x200
ProjectManager.KeepUserCode=true
ProjectManager.LastFirmware=true
ProjectManager.LibraryCopy=1
ProjectManager.MainLocation=Core/Src
ProjectManager.NoMain=false
ProjectManager.PreviousToolchain=
ProjectManager.ProjectBuild=false
ProjectManager.ProjectFileName=<PROJE_ADI>.ioc
ProjectManager.ProjectName=<PROJE_ADI>
ProjectManager.ProjectStructure=
ProjectManager.RegisterCallBack=
ProjectManager.StackSize=0x400
ProjectManager.TargetToolchain=STM32CubeIDE
ProjectManager.ToolChainLocation=
ProjectManager.UAScriptAfterPath=
ProjectManager.UAScriptBeforePath=
ProjectManager.UnderRoot=true
ProjectManager.functionlistsort=1-SystemClock_Config-RCC-false-HAL-false,2-MX_GPIO_Init-GPIO-false-HAL-true
<RCC_PARAMETRELERI>
VP_SYS_VS_Systick.Mode=SysTick
VP_SYS_VS_Systick.Signal=SYS_VS_Systick
board=Custom
boardIOC=false
isbadioc=false
```

#### RCC Parametre Tablosu — STM32F4 (HSI 16MHz kaynağından)

**25 MHz (HSI → PLL: M=8, N=50, P=DIV4, Q=7):**
```
RCC.48MHZClocksFreq_Value=14285714.285714285
RCC.AHBFreq_Value=25000000
RCC.APB1CLKDivider=RCC_HCLK_DIV4
RCC.APB1Freq_Value=6250000
RCC.APB1TimFreq_Value=12500000
RCC.APB2CLKDivider=RCC_HCLK_DIV2
RCC.APB2Freq_Value=12500000
RCC.APB2TimFreq_Value=25000000
RCC.CortexFreq_Value=25000000
RCC.EthernetFreq_Value=25000000
RCC.FCLKCortexFreq_Value=25000000
RCC.FamilyName=M
RCC.HCLKFreq_Value=25000000
RCC.HSE_VALUE=8000000
RCC.HSI_VALUE=16000000
RCC.I2SClocksFreq_Value=192000000
RCC.IPParameters=48MHZClocksFreq_Value,AHBFreq_Value,APB1CLKDivider,APB1Freq_Value,APB1TimFreq_Value,APB2CLKDivider,APB2Freq_Value,APB2TimFreq_Value,CortexFreq_Value,EthernetFreq_Value,FCLKCortexFreq_Value,FamilyName,HCLKFreq_Value,HSE_VALUE,HSI_VALUE,I2SClocksFreq_Value,LSE_VALUE,LSI_VALUE,MCO2PinFreq_Value,PLLCLKFreq_Value,PLLM,PLLN,PLLP,PLLQ,PLLQCLKFreq_Value,RTCFreq_Value,RTCHSEDivFreq_Value,SYSCLKFreq_VALUE,SYSCLKSource,VCOI2SOutputFreq_Value,VCOInputFreq_Value,VCOOutputFreq_Value,VcooutputI2S
RCC.LSE_VALUE=32768
RCC.LSI_VALUE=32000
RCC.MCO2PinFreq_Value=25000000
RCC.PLLCLKFreq_Value=25000000
RCC.PLLM=8
RCC.PLLN=50
RCC.PLLP=RCC_PLLP_DIV4
RCC.PLLQ=7
RCC.PLLQCLKFreq_Value=14285714.285714285
RCC.RTCFreq_Value=32000
RCC.RTCHSEDivFreq_Value=4000000
RCC.SYSCLKFreq_VALUE=25000000
RCC.SYSCLKSource=RCC_SYSCLKSOURCE_PLLCLK
RCC.VCOI2SOutputFreq_Value=384000000
RCC.VCOInputFreq_Value=2000000
RCC.VCOOutputFreq_Value=100000000
RCC.VcooutputI2S=192000000
```

**168 MHz — maksimum (HSI → PLL: M=8, N=168, P=DIV2, Q=7):**
```
RCC.48MHZClocksFreq_Value=48000000
RCC.AHBFreq_Value=168000000
RCC.APB1CLKDivider=RCC_HCLK_DIV4
RCC.APB1Freq_Value=42000000
RCC.APB1TimFreq_Value=84000000
RCC.APB2CLKDivider=RCC_HCLK_DIV2
RCC.APB2Freq_Value=84000000
RCC.APB2TimFreq_Value=168000000
RCC.CortexFreq_Value=168000000
RCC.EthernetFreq_Value=168000000
RCC.FCLKCortexFreq_Value=168000000
RCC.FamilyName=M
RCC.HCLKFreq_Value=168000000
RCC.HSE_VALUE=8000000
RCC.HSI_VALUE=16000000
RCC.I2SClocksFreq_Value=192000000
RCC.IPParameters=48MHZClocksFreq_Value,AHBFreq_Value,APB1CLKDivider,APB1Freq_Value,APB1TimFreq_Value,APB2CLKDivider,APB2Freq_Value,APB2TimFreq_Value,CortexFreq_Value,EthernetFreq_Value,FCLKCortexFreq_Value,FamilyName,HCLKFreq_Value,HSE_VALUE,HSI_VALUE,I2SClocksFreq_Value,LSE_VALUE,LSI_VALUE,MCO2PinFreq_Value,PLLCLKFreq_Value,PLLM,PLLN,PLLP,PLLQ,PLLQCLKFreq_Value,RTCFreq_Value,RTCHSEDivFreq_Value,SYSCLKFreq_VALUE,SYSCLKSource,VCOI2SOutputFreq_Value,VCOInputFreq_Value,VCOOutputFreq_Value,VcooutputI2S
RCC.LSE_VALUE=32768
RCC.LSI_VALUE=32000
RCC.MCO2PinFreq_Value=168000000
RCC.PLLCLKFreq_Value=168000000
RCC.PLLM=8
RCC.PLLN=168
RCC.PLLP=RCC_PLLP_DIV2
RCC.PLLQ=7
RCC.PLLQCLKFreq_Value=48000000
RCC.RTCFreq_Value=32000
RCC.RTCHSEDivFreq_Value=4000000
RCC.SYSCLKFreq_VALUE=168000000
RCC.SYSCLKSource=RCC_SYSCLKSOURCE_PLLCLK
RCC.VCOI2SOutputFreq_Value=384000000
RCC.VCOInputFreq_Value=2000000
RCC.VCOOutputFreq_Value=336000000
RCC.VcooutputI2S=192000000
```

**180 MHz — STM32F446 maksimum (HSI → PLL: M=8, N=180, P=DIV2, Q=4):**
```
RCC.48MHZClocksFreq_Value=45000000
RCC.AHBFreq_Value=180000000
RCC.APB1CLKDivider=RCC_HCLK_DIV4
RCC.APB1Freq_Value=45000000
RCC.APB1TimFreq_Value=90000000
RCC.APB2CLKDivider=RCC_HCLK_DIV2
RCC.APB2Freq_Value=90000000
RCC.APB2TimFreq_Value=180000000
RCC.CortexFreq_Value=180000000
RCC.FCLKCortexFreq_Value=180000000
RCC.FamilyName=M
RCC.HCLKFreq_Value=180000000
RCC.HSE_VALUE=8000000
RCC.HSI_VALUE=16000000
RCC.IPParameters=48MHZClocksFreq_Value,AHBFreq_Value,APB1CLKDivider,APB1Freq_Value,APB1TimFreq_Value,APB2CLKDivider,APB2Freq_Value,APB2TimFreq_Value,CortexFreq_Value,FCLKCortexFreq_Value,FamilyName,HCLKFreq_Value,HSE_VALUE,HSI_VALUE,LSE_VALUE,LSI_VALUE,MCO2PinFreq_Value,PLLCLKFreq_Value,PLLM,PLLN,PLLP,PLLQ,PLLQCLKFreq_Value,RTCFreq_Value,RTCHSEDivFreq_Value,SYSCLKFreq_VALUE,SYSCLKSource,VCOInputFreq_Value,VCOOutputFreq_Value
RCC.LSE_VALUE=32768
RCC.LSI_VALUE=32000
RCC.MCO2PinFreq_Value=180000000
RCC.PLLCLKFreq_Value=180000000
RCC.PLLM=8
RCC.PLLN=180
RCC.PLLP=RCC_PLLP_DIV2
RCC.PLLQ=4
RCC.PLLQCLKFreq_Value=45000000
RCC.RTCFreq_Value=32000
RCC.RTCHSEDivFreq_Value=4000000
RCC.SYSCLKFreq_VALUE=180000000
RCC.SYSCLKSource=RCC_SYSCLKSOURCE_PLLCLK
RCC.VCOInputFreq_Value=2000000
RCC.VCOOutputFreq_Value=360000000
```

> Tabloda olmayan bir clock için: VCOInput = HSI/PLLM, VCOOutput = VCOInput * PLLN, SYSCLK = VCOOutput / PLLP. STM32F4 kısıtları: VCOInput 1–2 MHz, VCOOutput 100–432 MHz, APB1 ≤ 42 MHz, APB2 ≤ 84 MHz.

---

### Core/Src/main.c

`WID/Examples/Src/main.c` yapısını birebir takip et. `SystemClock_Config()` clock hızına göre doldur:

**FLASH_LATENCY seçim tablosu (STM32F4):**

| SYSCLK       | FLASH_LATENCY       | PLLM | PLLN | PLLP       |
|--------------|---------------------|------|------|------------|
| ≤ 30 MHz     | FLASH_LATENCY_0     | 8    | 50   | DIV4       |
| ≤ 60 MHz     | FLASH_LATENCY_1     | 8    | 60   | DIV2 → 60M |
| ≤ 90 MHz     | FLASH_LATENCY_2     | 8    | 90   | DIV2       |
| ≤ 120 MHz    | FLASH_LATENCY_3     | 8    | 120  | DIV2       |
| ≤ 150 MHz    | FLASH_LATENCY_4     | 8    | 150  | DIV2       |
| ≤ 168 MHz    | FLASH_LATENCY_5     | 8    | 168  | DIV2       |
| ≤ 180 MHz    | FLASH_LATENCY_5     | 8    | 180  | DIV2       |

`SystemClock_Config()` içeriği (168 MHz örneği — clock'a göre değerleri adapte et):

```c
void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  /** Configure the main internal regulator output voltage
  */
  __HAL_RCC_PWR_CLK_ENABLE();
  __HAL_PWR_VOLTAGESCALING_CONFIG(PWR_REGULATOR_VOLTAGE_SCALE1);

  /** Initializes the RCC Oscillators according to the specified parameters
  * in the RCC_OscInitTypeDef structure.
  */
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI;
  RCC_OscInitStruct.PLL.PLLM = 8;
  RCC_OscInitStruct.PLL.PLLN = 168;
  RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;
  RCC_OscInitStruct.PLL.PLLQ = 7;
  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)
  {
    Error_Handler();
  }

  /** Initializes the CPU, AHB and APB buses clocks
  */
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5) != HAL_OK)
  {
    Error_Handler();
  }
}
```

Dosya yapısı (tam sıra):
1. `/* USER CODE BEGIN Header */` … `/* USER CODE END Header */`
2. `#include "main.h"`
3. Bölüm ayraçları: `/* Private includes ---*/`, `/* Private typedef ---*/`, `/* Private define ---*/`, `/* Private macro ---*/`, `/* Private variables ---*/`
4. `void SystemClock_Config(void);` prototipi
5. `/* Private function prototypes ---*/`
6. `/* Private user code ---*/` → `/* USER CODE BEGIN 0 */` … `/* USER CODE END 0 */`
7. `int main(void)` → `HAL_Init()` → `SystemClock_Config()` → while(1)
8. `SystemClock_Config()` implementasyonu
9. `/* USER CODE BEGIN 4 */` … `/* USER CODE END 4 */`
10. `Error_Handler()` ve `#ifdef USE_FULL_ASSERT` bloku

---

### Core/Inc/main.h

```c
/* USER CODE BEGIN Header */
/**
  ******************************************************************************
  * @file           : main.h
  * @brief          : Header for main.c file.
  *                   This file contains the common defines of the application.
  ******************************************************************************
  * @attention
  *
  * Copyright (c) <YIL> STMicroelectronics.
  * All rights reserved.
  *
  * This software is licensed under terms that can be found in the LICENSE file
  * in the root directory of this software component.
  * If no LICENSE file comes with this software, it is provided AS-IS.
  *
  ******************************************************************************
  */
/* USER CODE END Header */

/* Define to prevent recursive inclusion -------------------------------------*/
#ifndef __MAIN_H
#define __MAIN_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
#include "stm32<aile>xx_hal.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

/* USER CODE END Includes */

/* Exported types ------------------------------------------------------------*/
/* USER CODE BEGIN ET */

/* USER CODE END ET */

/* Exported constants --------------------------------------------------------*/
/* USER CODE BEGIN EC */

/* USER CODE END EC */

/* Exported macro ------------------------------------------------------------*/
/* USER CODE BEGIN EM */

/* USER CODE END EM */

/* Exported functions prototypes ---------------------------------------------*/
void Error_Handler(void);

/* USER CODE BEGIN EFP */

/* USER CODE END EFP */

/* Private defines -----------------------------------------------------------*/

/* USER CODE BEGIN Private defines */

/* USER CODE END Private defines */

#ifdef __cplusplus
}
#endif

#endif /* __MAIN_H */
```

---

### Core/Src/stm32\<aile\>xx_it.c

`WID/Examples/Src/stm32f4xx_it.c` yapısını birebir takip et. Zorunlu handler'lar:
- `NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler`, `UsageFault_Handler`
- `SVC_Handler`, `DebugMon_Handler`, `PendSV_Handler`
- `SysTick_Handler` → içinde `HAL_IncTick()` çağrısı
- Her handler'da `USER CODE BEGIN/END` marker'ları

---

### Core/Inc/stm32\<aile\>xx_it.h

`WID/Examples/Inc/stm32f4xx_it.h` yapısını takip et:
- Include guard: `#ifndef __STM32<AİLE>XX_IT_H`
- `extern "C"` blokları
- Tüm handler prototipleri

---

### Core/Src/stm32\<aile\>xx_hal_msp.c

`WID/Examples/Src/stm32f4xx_hal_msp.c` yapısını takip et:
- `HAL_MspInit()` fonksiyonu
- `__HAL_RCC_SYSCFG_CLK_ENABLE()` ve `__HAL_RCC_PWR_CLK_ENABLE()`
- `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_0)`
- `USER CODE BEGIN/END` marker'ları

---

### Core/Src/syscalls.c ve sysmem.c

`WID/Examples/Src/syscalls.c` ve `sysmem.c` dosyalarını oku ve birebir kopyala. Bu dosyalar MCU bağımsız newlib stub'larıdır; içerikleri değişmez.

---

## Adım 4: Proje Yönetim Dosyaları Oluştur

### CLAUDE.md
`~/.claude/templates/claude-stm32.md` dosyasını oku ve `<proje_adı>/CLAUDE.md` olarak yaz.
Dosya bulunamazsa: "⚠️ Şablon bulunamadı: `~/.claude/templates/claude-stm32.md`"

### PROJECT.md
```markdown
# Proje: <proje_adı>

## Donanım
- **MCU:** <mcu_modeli>
- **Sistem Clock:** <clock_frekansı>
- **Flash:** <flash_boyutu>
- **RAM:** <ram_boyutu>

## Proje Hedefi
<kullanıcının verdiği açıklama>

## RTOS
<FreeRTOS kullanılıyor / kullanılmıyor>

## Oluşturulma
<tarih>
```

### STATE.md
```markdown
# Proje Durumu

## Genel
- **Aktif Proje:** <proje_adı>
- **MCU:** <mcu>
- **Son Güncelleme:** <tarih>

## Görev Durumu
- **active_prp:** null
- **failure_count:** 0
- **gate5_pending:** false
- **gate5_prp:** null
- **git_language:** null

## Blokajlar
(yok)

## Son Oturum Özeti
Proje oluşturuldu.
```

### REQUIREMENTS.md
```markdown
# Gereksinimler: <proje_adı>

## Fonksiyonel Gereksinimler
<kullanıcının verdiği gereksinim listesi — her satır maddeli>

## Kısıtlar
- Flash: <flash_boyutu>
- RAM: <ram_boyutu>
- Clock: <clock_frekansı>
```

### ROADMAP.md
```markdown
# Yol Haritası: <proje_adı>

## Faz 1: Temel Altyapı
- [ ] Clock tree konfigürasyonu
- [ ] GPIO init
- [ ] Debug UART (console)
**Tamamlanma Kriteri:** LED yanıp sönüyor, UART'tan "Hello" geliyor.

## Faz 2: Peripheral Sürücüleri
- [ ] <kullanıcının ihtiyacına göre sürücüler>
**Tamamlanma Kriteri:** Her peripheral loopback testi geçiyor.

## Faz 3: Uygulama Katmanı
- [ ] <uygulama modülleri>
**Tamamlanma Kriteri:** Tüm gereksinimler karşılanıyor.
```

### project_memory/hardware.md
```markdown
# Donanım Notları

## MCU Bilgileri
- **Model:** <mcu_modeli>
- **Core:** Cortex-M<x>
- **Flash:** <flash>
- **RAM:** <ram>
- **Clock (max):** <max_clock>

## Konfigürasyon
- **Sistem Clock:** <clock_frekansı>
- **Clock Kaynağı:** HSI (dahili 16 MHz)
- **PLL:** M=<pllm>, N=<plln>, P=<pllp>

## Pin Atamaları
(Peripheral'lar eklendikçe güncellenecek)

## Donanım Kısıtları
(Keşfedildikçe eklenecek)
```

### project_memory/decisions.md, bugs.md, progress.md, changelog.md

`decisions.md`: `# Mimari Kararlar\n(Henüz kayıt yok)`
`bugs.md`: `# Bug Kayıtları\n(Henüz kayıt yok)`
`progress.md`: Faz 1 görevlerini "Yapılacaklar" olarak listele
`changelog.md`: `[başlangıç]` girdisi ile proje oluşturma kaydı

---

## Adım 5: Kullanıcıya Bildir

```
✓ Proje oluşturuldu: <proje_adı>
  MCU: <mcu> | Clock: <clock> | Flash: <flash> | RAM: <ram>

CubeIDE Dosyaları:
  .project
  <proje_adı>.ioc
  Core/Src/main.c
  Core/Src/stm32<aile>xx_it.c, stm32<aile>xx_hal_msp.c
  Core/Src/syscalls.c, sysmem.c
  Core/Inc/main.h, stm32<aile>xx_it.h

Proje Yönetim Dosyaları:
  CLAUDE.md, PROJECT.md, STATE.md, REQUIREMENTS.md, ROADMAP.md
  project_memory/ (hardware, decisions, bugs, progress, changelog)

CubeIDE'de Açmak ve Karta Yüklemek İçin:
  1. STM32CubeIDE → File → Open Projects from File System
  2. <proje_adı>/ klasörünü seç → Finish
  3. <proje_adı>.ioc dosyasına çift tıkla
  4. "Generate Code" butonuna bas
     → HAL driver dosyaları, linker script ve startup dosyası otomatik indirilir
  5. Project → Build All (Ctrl+B)
  6. Run → Debug / Flash

Sonraki adım: /generate-prp ile ilk görevinizi tanımlayın.
Örnek: /generate-prp "Temel GPIO ve LED blink implementasyonu"
```
