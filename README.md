# Crazyflie firmware: WS2812 на PB8 / TIM4_CH3

Модифицированная прошивка STM32 для квадрокоптера **Crazyflie 2.1**, выполненная в рамках дипломного проекта бакалавра Фан Ин Су(РТУ МИРЭА).

Основное изменение: вывод управления светодиодной лентой **WS2812** перенесён с `PB5` / `TIM3_CH2` на **`PB8` / `TIM4_CH3`** (контакт IO1 разъёма деки).

> **Базовая версия:** [bitcraze/crazyflie-firmware](https://github.com/bitcraze/crazyflie-firmware), тег **`2025.09`**.
> Все изменения — один коммит поверх этого тега. Посмотреть их целиком можно через сравнение `2025.09...main`.

---

## Содержание

1. [Обзор изменений](#обзор-изменений)
2. [Подробное описание изменений](#подробное-описание-изменений)
3. [Требования](#требования)
4. [Клонирование и сборка](#клонирование-и-сборка)
5. [Прошивка](#прошивка)
6. [Симуляция (Crazyswarm2)](#симуляция-crazyswarm2)
7. [Ссылки](#ссылки)

---

## Обзор изменений

| Параметр | Было (2025.09) | Стало |
|---|---|---|
| Пин данных WS2812 | `PB5` | **`PB8`** (IO1) |
| Таймер / канал | `TIM3_CH2` (`CCR2`) | **`TIM4_CH3`** (`CCR3`) |
| DMA | `DMA1_Stream5`, `DMA_Channel_5` | **`DMA1_Stream7`, `DMA_Channel_2`** |
| Обработчик прерывания | `DMA1_Stream5_IRQHandler` | **`DMA1_Stream7_IRQHandler`** |
| Ресурсы деки LED-ring | `DECK_USING_TIMER3` | **`DECK_USING_TIMER4`** |

Изменённые файлы:

| Файл | Суть изменения |
|---|---|
| `src/drivers/src/ws2812_cf2.c` | Перенос генерации сигнала WS2812 на PB8 / TIM4_CH3 / DMA1_Stream7 |
| `src/deck/drivers/src/ledring12.c` | Обновлена карта ресурсов драйвера деки (таймер и GPIO) |
| `src/deck/core/deck_info.c` | Отключена проверка конфликтов периферии и GPIO между деками |
| `src/deck/drivers/src/Kconfig` | Настройки драйверов дек |
| `Makefile` | Принудительно задан тип платы `CF21` |

---

## Подробное описание изменений

### `src/drivers/src/ws2812_cf2.c`

Сигнал WS2812 формируется аппаратно: ШИМ таймера с частотой 800 кГц, значения скважности для каждого бита подаются в регистр сравнения таймера по DMA. Это позволяет выдерживать жёсткие тайминги протокола WS2812 без нагрузки на CPU. Логика драйвера не менялась, перенесены только используемые аппаратные ресурсы.

**1. GPIO и альтернативная функция (строки ~71–89)**

```c
RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM4, ENABLE);        // было: TIM3

GPIO_InitStructure.GPIO_Pin = GPIO_Pin_8;                   // было: GPIO_Pin_5
GPIO_PinAFConfig(GPIOB, GPIO_PinSource8, GPIO_AF_TIM4);     // было: PinSource5, GPIO_AF_TIM3
```

`PB8` аппаратно соответствует выходу `TIM4_CH3` на STM32F405.

**2. Таймер: TIM4, канал 3, режим PWM1 (строки ~98–109)**

```c
TIM_TimeBaseInit(TIM4, &TIM_TimeBaseStructure);
TIM_OC3Init(TIM4, &TIM_OCInitStructure);                    // было: TIM_OC2Init(TIM3, ...)
TIM_OC3PreloadConfig(TIM4, TIM_OCPreload_Enable);
TIM_CtrlPWMOutputs(TIM4, ENABLE);
```

Частота (800 кГц) и режим PWM1 сохранены.

**3. DMA и NVIC (строки ~111–149)**

```c
DMA_DeInit(DMA1_Stream7);
DMA_InitStructure.DMA_Channel = DMA_Channel_2;
DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&TIM4->CCR3;
DMA_Init(DMA1_Stream7, &DMA_InitStructure);

NVIC_InitStructure.NVIC_IRQChannel = DMA1_Stream7_IRQn;

TIM_DMACmd(TIM4, TIM_DMA_CC3, ENABLE);
```

По карте DMA-запросов STM32F4 запрос `TIM4_CH3` доступен на `DMA1_Stream7 / Channel 2`.

Сначала использовался `DMA1_Stream2`, но его обработчик `DMA1_Stream2_IRQHandler` уже определён в `i2c_drv.c`, из-за чего возникала ошибка линковки (двойное определение). Поэтому выбран свободный `DMA1_Stream7`.

**4. Запуск передачи, `ws2812Send` (строки ~203–205)**

```c
DMA1_Stream7->NDTR = sizeof(led_dma.buffer) / sizeof(led_dma.buffer[0]);
DMA_Cmd(DMA1_Stream7, ENABLE);
TIM_Cmd(TIM4, ENABLE);                                      // было: TIM3
```

**5. Обработчик DMA, `ws2812DmaIsr` (строки ~216–247)**

- обработка прерываний половины и окончания передачи: `DMA_IT_HTIF7` / `DMA_IT_TCIF7` (было `HTIF5` / `TCIF5`) — дозаполнение соответствующей половины двойного буфера;
- по окончании передачи (и при `total_led == 0`) останавливаются `TIM4` и `DMA1_Stream7`.

**6. Обработчик прерывания (строки ~253–256)**

```c
#if !defined(CONFIG_DECK_USD_USE_ALT_PINS_AND_SPI) && !defined(CONFIG_MOTORS_ESC_PROTOCOL_DSHOT)
void __attribute__((used)) DMA1_Stream7_IRQHandler(void)
{
  ws2812DmaIsr();
}
#endif
```

Обёртка `#if` сохранена из оригинала: при включённом протоколе DSHOT обработчик `DMA1_Stream7` реализован в `motors.c`, и этот код не компилируется, чтобы избежать конфликта.

> ⚠️ Следствие: **WS2812 на PB8 несовместим с DSHOT** (`CONFIG_MOTORS_ESC_PROTOCOL_DSHOT`) и с альтернативными пинами microSD-деки (`CONFIG_DECK_USD_USE_ALT_PINS_AND_SPI`), так как они используют тот же DMA-поток.

### `src/deck/drivers/src/ledring12.c`

Обновлено описание ресурсов в структуре драйвера деки (`ledring12_deck`, конец файла), чтобы оно соответствовало фактическому использованию:

```c
.usedPeriph = DECK_USING_TIMER4,                    // было: DECK_USING_TIMER3
.usedGpio   = DECK_USING_IO_1 | DECK_USING_IO_3,    // было: DECK_USING_IO_2 | DECK_USING_IO_3
```

| Контакт деки | Пин MCU | Назначение |
|---|---|---|
| IO1 | `PB8` | данные WS2812 (TIM4_CH3) |
| IO3 | `PB4` | обычный GPIO — фары (`setHeadlightsOn()`), без изменений |
| IO2 | `PB5` | больше не используется |

### `src/deck/core/deck_info.c`

Закомментирован код, связанный с `checkPeriphAndGpioConflicts`, — проверка конфликтов ресурсов между подключёнными деками отключена.

> ⚠️ При одновременном подключении нескольких дек прошивка не предупредит о конфликте по таймерам и пинам — совместимость нужно контролировать вручную.

### `src/deck/drivers/src/Kconfig`

Изменены настройки драйверов дек (4 строки). Драйвер LED-ring загружается принудительно через параметр **Deck Force** в `menuconfig` (см. [Сборка](#клонирование-и-сборка)), так как самодельная лента не имеет EEPROM-идентификатора деки.

### `Makefile`

```make
ARCH_CFLAGS += -DDEVICE_TYPE_STRING_FORCE="CF21"
```

Принудительно задаёт тип платы `CF21`, чтобы прошивка не зависела от типа устройства, записанного в OTP-памяти.

---

## Требования

Проверено на **Ubuntu 24.04**.

```bash
sudo apt update
sudo apt install git make gcc-arm-none-eabi python3-venv dfu-util openocd
```

Python-окружение для работы с Crazyflie:

```bash
python3 -m venv crazy
source crazy/bin/activate
pip install cflib cfclient
```

Оборудование:

- Crazyflie 2.1;
- Crazyradio PA — для прошивки по радио;
- USB-кабель — для прошивки через DFU;
- ST-Link V2 и debug-адаптер Crazyflie — для прошивки по SWD (необязательно).

---

## Клонирование и сборка

Клонировать **обязательно с подмодулями** — без `vendor/` прошивка не соберётся:

```bash
git clone --recursive https://github.com/VitalyFan/crazyflie-mirea.git
cd crazyflie-mirea
```

Если репозиторий уже склонирован без `--recursive`:

```bash
git submodule update --init --recursive
```

Конфигурация и сборка:

```bash
make cf2_defconfig
make menuconfig
make -j$(nproc)
```

В `menuconfig` необходимо включить принудительную загрузку драйвера LED-ring:

```
Expansion deck configuration  --->
    Force load specified custom deck driver  →  bcLedRing
```

Результат сборки: `build/cf2.bin`.

> Файл `.config`, создаваемый `menuconfig`, не хранится в Git. На новом компьютере настройку нужно выполнить заново.

---

## Прошивка

Карта памяти STM32F405 в Crazyflie:

| Адрес | Содержимое |
|---|---|
| `0x08000000` | загрузчик STM32 (`cf2loader.bin`) |
| `0x08004000` | основная прошивка (`cf2.bin`) |

### Способ 1. По радио через Crazyradio (рекомендуется)

1. Выключенный Crazyflie: зажать кнопку питания примерно на 3 секунды, пока синие светодиоды не начнут мигать — дрон в режиме загрузчика.
2. Подключить Crazyradio к компьютеру и выполнить:

```bash
make cload
```

### Способ 2. По USB через DFU

Подходит, если прошивка по радио недоступна.

1. Отключить аккумулятор.
2. Зажать кнопку питания и, не отпуская, подключить USB-кабель. Удерживать около 5 секунд, затем отпустить.
3. Проверить, что устройство в режиме DFU:

```bash
dfu-util -l        # должно появиться устройство 0483:df11
```

4. Записать прошивку:

```bash
dfu-util -d 0483:df11 -a 0 -s 0x08004000:leave -D build/cf2.bin
```

### Способ 3. Через ST-Link (SWD)

```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
  -c "program build/cf2.bin verify reset exit 0x08004000"
```

> ⚠️ Основную прошивку записывать по адресу **`0x08004000`**. Запись `cf2.bin` по адресу `0x08000000` затрёт загрузчик.

### Восстановление загрузчика STM32

Нужно, только если загрузчик повреждён. Исходники: [crazyflie2-stm-bootloader](https://github.com/bitcraze/crazyflie2-stm-bootloader).

```bash
git clone https://github.com/bitcraze/crazyflie2-stm-bootloader
```

Запись через DFU:

```bash
dfu-util -d 0483:df11 -a 0 -s 0x08000000 -D cf2loader.bin
```

Или через ST-Link:

```bash
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
  -c "init; reset halt; flash write_image erase cf2loader.bin 0x08000000; reset; exit"
```

### Восстановление nRF51 (контроллер питания и радио)

Нужно, только если прошивка nRF51 повреждена. Выполняется через ST-Link, подключённый к SWD-выводам nRF51. Исходники: [crazyflie2-nrf-firmware](https://github.com/bitcraze/crazyflie2-nrf-firmware).

```bash
git clone https://github.com/bitcraze/crazyflie2-nrf-firmware
cd crazyflie2-nrf-firmware
```

Все команды ниже выполняются из этого каталога.

**1. Полная очистка flash-памяти:**

```bash
openocd -f interface/stlink.cfg -f target/nrf51.cfg \
  -c "adapter speed 100" \
  -c "init" \
  -c "halt" \
  -c "nrf51 mass_erase" \
  -c "shutdown"
```

**2. Запись SoftDevice S130:**

```bash
openocd -f interface/stlink.cfg -c "transport select hla_swd" -f target/nrf51.cfg \
  -c "init" -c "reset halt" \
  -c "flash write_image erase $(pwd)/vendor/nrf5sdk/components/softdevice/s130/hex/s130_nrf51_2.0.1_softdevice.hex" \
  -c "reset" -c "exit"
```

**3. Запись MBS (master boot switch) и регистров UICR:**

```bash
openocd -f interface/stlink.cfg -c "transport select hla_swd" -f target/nrf51.cfg \
  -c "init" -c "reset halt" \
  -c "mww 0x4001e504 0x01" \
  -c "flash write_image erase $(pwd)/nrf_mbs.hex" \
  -c "verify_image $(pwd)/nrf_mbs.hex" \
  -c "mww 0x4001e504 0x01" \
  -c "mww 0x10001014 0x3F000" \
  -c "mww 0x10001080 0x3A000" \
  -c "reset run" -c "exit"
```

**4. Сборка и запись прошивки nRF51:**

```bash
make
openocd -f interface/stlink.cfg -f target/nrf51.cfg \
  -c "adapter speed 100" \
  -c "init" \
  -c "halt" \
  -c "wait_halt 5000" \
  -c "flash write_image erase _build/cf2_nrf.hex" \
  -c "verify_image _build/cf2_nrf.hex" \
  -c "shutdown"
```

---

## Симуляция (Crazyswarm2)

Необязательный раздел. Требуется ROS 2 Jazzy и установленный Crazyswarm2.

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

Сборка Python-биндингов прошивки (используются бэкендом симуляции):

```bash
make cf2_defconfig
make bindings_python
export PYTHONPATH=$PYTHONPATH:$(pwd)/build
```

Запуск симуляции в двух терминалах:

```bash
# терминал 1
ros2 launch crazyflie launch.py backend:=sim

# терминал 2
ros2 run crazyflie_examples hello_world --ros-args -p use_sim_time:=True
```

Генерация траектории с помощью [uav_trajectories](https://github.com/whoenig/uav_trajectories):

```bash
python3 uav_trajectories/scripts/generate_trajectory.py spiral_timed_0.csv spiral_traj_0.csv --pieces 5
```

---

## Ссылки

- [bitcraze/crazyflie-firmware](https://github.com/bitcraze/crazyflie-firmware) — оригинальная прошивка
- [Обсуждение #2140](https://github.com/orgs/bitcraze/discussions/2140) и [#2141](https://github.com/orgs/bitcraze/discussions/2141) на форуме Bitcraze
- [Документация Crazyflie](https://www.bitcraze.io/documentation/)

## Лицензия

Прошивка распространяется под лицензией **GPL-3.0**, как и оригинальный проект Bitcraze (см. файл `LICENSE`).
