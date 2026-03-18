# STM32 Gömülü Sistem Kodlama Standartları

> Bu dosya yalnızca STM32 kodlama standartlarını içerir.
> Master ajanı başlatmak için: `/embedded-master` komutunu çalıştır.

## Zorunlu Kodlama Kuralları

- Tüm integer'lar explicit-width type kullanır: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t` (`int`, `long` yasak)
- Tüm HAL çağrıları dönüş değeri kontrol edilir: `if (HAL_UART_Init(&huart1) != HAL_OK) { Error_Handler(); }`
- ISR ve main arasında paylaşılan tüm değişkenler `volatile` zorunlu: `volatile uint32_t flag;`
- ISR içinde dinamik bellek tahsisi (`malloc`/`free`) kesinlikle yasak
- Include guard zorunlu — `#pragma once` değil:
  ```c
  #ifndef MY_MODULE_H
  #define MY_MODULE_H
  // ...
  #endif /* MY_MODULE_H */
  ```
- Max 300 satır per `.c` dosyası — aşılırsa `_init.c`, `_config.c`, `_runtime.c`, `_irq.c` modüllerine böl
- Cortex-M7 (STM32H7 serisi) DMA transferleri öncesi ve sonrası cache yönetimi zorunlu:
  ```c
  SCB_CleanDCacheByAddr((uint32_t*)tx_buf, sizeof(tx_buf));   // TX öncesi
  SCB_InvalidateDCacheByAddr((uint32_t*)rx_buf, sizeof(rx_buf)); // RX sonrası
  ```
- CubeMX üretilen referans konfigürasyonuyla karşılaştırma zorunlu
- Donanım konfigürasyonları (register map, pin atamaları, clock tree) YAML formatında belgelenir

## Dosya Organizasyonu

```
src/
  <peripheral>_init.c      # Peripheral başlatma (HAL_XXX_Init)
  <peripheral>_config.c    # Konfigürasyon yardımcıları
  <peripheral>_runtime.c   # Normal çalışma fonksiyonları
  <peripheral>_irq.c       # Interrupt handler'lar
inc/
  <peripheral>.h           # Public API (include guard zorunlu)
```

## Oturum Başlangıcı (Master Ajan Aktifken)

1. Bu `CLAUDE.md` dosyasını oku
2. `STATE.md` oku (`failure_count`, `active_prp`, son durum)
3. `project_memory/*.md` dosyalarını oku
4. Kullanıcıya bildir: `"Aktif proje: <proje> | MCU: <mcu> | Durum: <özet>"`
