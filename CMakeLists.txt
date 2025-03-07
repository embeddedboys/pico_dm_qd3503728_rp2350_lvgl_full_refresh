# Copyright (c) 2024 embeddedboys developers

# Permission is hereby granted, free of charge, to any person obtaining
# a copy of this software and associated documentation files (the
# "Software"), to deal in the Software without restriction, including
# without limitation the rights to use, copy, modify, merge, publish,
# distribute, sublicense, and/or sell copies of the Software, and to
# permit persons to whom the Software is furnished to do so, subject to
# the following conditions:

# The above copyright notice and this permission notice shall be
# included in all copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
# EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
# MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
# NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
# LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
# OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
# WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

# This is the main CMakeLists file of the project. It is used to build the project
# using the CMake build system. You can find the documentation of CMake at
# https:#cmake.org/cmake/help/latest/index.html

cmake_minimum_required(VERSION 3.13)

# initialize the SDK based on PICO_SDK_PATH
# note: this must happen before project()
include(pico_sdk_import.cmake)
include(pico_extras_import_optional.cmake)

# set the project name
project(pico_dm_qd3503728 C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
SET(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -Wl,--print-memory-usage")
SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -Wl,--print-memory-usage")

if (PICO_SDK_VERSION_STRING VERSION_LESS "2.0.0")
    message(FATAL_ERROR "Raspberry Pi Pico SDK version 2.0.0 (or later) required. Your version is ${PICO_SDK_VERSION_STRING}")
endif()

# If you want debug output from USB (pass -DPICO_STDIO_USB=1) this ensures you don't lose any debug output while USB is set up
if (NOT DEFINED PICO_STDIO_USB_CONNECT_WAIT_TIMEOUT_MS)
    set(PICO_STDIO_USB_CONNECT_WAIT_TIMEOUT_MS 3000)
endif()

# initialize the Raspberry Pi Pico SDK
pico_sdk_init()

add_compile_options(-Wall
        -Wno-format          # int != int32_t as far as the compiler is concerned because gcc has int32_t as long int
        -Wno-unused-function # we have some for the docs that aren't called
        )
if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-Wno-maybe-uninitialized)
endif()

# Set all global variables here
set(OVERCLOCK_ENABLED 0)    # 1: enable, 0: disable

if(OVERCLOCK_ENABLED)

    message(WARNING "Overclocking is enabled. This may damage your device. Use at own risk.")

    if(${PICO_BOARD} STREQUAL "pico" OR ${PICO_PLATFORM} STREQUAL "rp2040")
        message(FATAL_ERROR "This demo does not support the pico board.")
    elseif(${PICO_BOARD} STREQUAL "pico2" OR ${PICO_PLATFORM} STREQUAL "rp2350")
        # Overclocking profiles
        #      SYS_CLK  | FLASH_CLK | Voltage
        #  1  | 366MHz  |  122MHz   |  1.20(V)
        #  2  | 384MHz  |  128MHz   |  1.20(V)
        set(OVERCLOCK_PROFILE 2)

        if(OVERCLOCK_PROFILE EQUAL 1)
            set(SYS_CLK_KHZ 366000)             # CPU clock speed
            set(PERI_CLK_KHZ ${SYS_CLK_KHZ})    # Peripheral clock speed
        elseif(OVERCLOCK_PROFILE EQUAL 2)
            set(SYS_CLK_KHZ 384000)             # CPU clock speed
            set(PERI_CLK_KHZ ${SYS_CLK_KHZ})    # Peripheral clock speed
        else()
            message(FATAL_ERROR "Invalid overclocking profile")
        endif()
    endif()

else()  # OVERCLOCK_ENABLED
    message(WARNING "Overclocking is disabled.")

    if(${PICO_BOARD} STREQUAL "pico" OR ${PICO_PLATFORM} STREQUAL "rp2040")
        message(FATAL_ERROR "This demo does not support the pico board.")
    elseif(${PICO_BOARD} STREQUAL "pico2" OR ${PICO_PLATFORM} STREQUAL "rp2350")
        set(SYS_CLK_KHZ 150000) # CPU clock speed
        set(PERI_CLK_KHZ ${SYS_CLK_KHZ})    # Peripheral clock speed
    endif()

endif() # OVERCLOCK_ENABLED

# LCD Pins for 8080 interface
set(LCD_PIN_DB_BASE  0)  # 8080 LCD data bus base pin
set(LCD_PIN_DB_COUNT 16) # 8080 LCD data bus pin count
set(LCD_PIN_CS  18)  # 8080 LCD chip select pin
set(LCD_PIN_WR  19)  # 8080 LCD write pin
set(LCD_PIN_RS  20)  # 8080 LCD register select pin
set(LCD_PIN_RST 22)  # 8080 LCD reset pin
set(LCD_PIN_BL  28)  # 8080 LCD backlight pin
set(LCD_HOR_RES 480)
set(LCD_VER_RES 320)
set(DISP_OVER_PIO 1) # 1: PIO, 0: GPIO
set(PIO_USE_DMA   1)   # 1: use DMA, 0: not use DMA
if(OVERCLOCK_ENABLED)
    set(I80_BUS_WR_CLK_KHZ 58000)
else()
    set(I80_BUS_WR_CLK_KHZ 50000)
endif()

# Rotation configuration
set(LCD_ROTATION 1)  # 0: normal, 1: 90 degree, 2: 180 degree, 3: 270 degree
if(${LCD_ROTATION} EQUAL 0 OR ${LCD_ROTATION} EQUAL 2)
    set(LCD_HOR_RES 320)
    set(LCD_VER_RES 480)
elseif(${LCD_ROTATION} EQUAL 1 OR ${LCD_ROTATION} EQUAL 3)
    set(LCD_HOR_RES 480)
    set(LCD_VER_RES 320)
else()
    message(FATAL_ERROR "ERROR: Invalid Display rotation")
endif()

# Display buffer size configuration
if(${PICO_BOARD} STREQUAL "pico" OR ${PICO_PLATFORM} STREQUAL "rp2040")
    message(FATAL_ERROR "This demo does not support the pico board.")
elseif(${PICO_BOARD} STREQUAL "pico2" OR ${PICO_PLATFORM} STREQUAL "rp2350")
    math(EXPR MY_DISP_BUF_SIZE "${LCD_HOR_RES} * ${LCD_VER_RES}")
endif()

include_directories(./ include)

# add lvgl library here
add_subdirectory(lvgl)

# include PIO library here
add_subdirectory(pio)
target_compile_definitions(pio_i80 PUBLIC LCD_PIN_RS=${LCD_PIN_RS})
target_compile_definitions(pio_i80 PUBLIC LCD_PIN_CS=${LCD_PIN_CS})
target_compile_definitions(pio_i80 PUBLIC DEFAULT_PIO_CLK_KHZ=${PERI_CLK_KHZ})
target_compile_definitions(pio_i80 PUBLIC PIO_USE_DMA=${PIO_USE_DMA})
target_compile_definitions(pio_i80 PUBLIC I80_BUS_WR_CLK_KHZ=${I80_BUS_WR_CLK_KHZ})

# include factory test library here
# add_subdirectory(factory)

# lv_conf.h need pico header files e.g. the custom tick
target_link_libraries(lvgl PRIVATE pico_stdlib)

# user define common source files
file(GLOB_RECURSE COMMON_SOURCES
    main.c
    ili9488.c
    ft6236.c
    porting/lv_port_disp_template.c
    porting/lv_port_indev_template.c
    i2c_tools.c
    backlight.c
)

# rest of your project
add_executable(${PROJECT_NAME} ${COMMON_SOURCES})
target_link_libraries(${PROJECT_NAME}
    pico_bootsel_via_double_reset
    pico_stdlib
    hardware_pwm
    hardware_i2c
    pio_i80
    # factory_test
    lvgl lvgl::demos lvgl::examples
)
target_include_directories(${PROJECT_NAME} PUBLIC .)

# add target common defines here
target_compile_definitions(${PROJECT_NAME} PUBLIC DEFAULT_SYS_CLK_KHZ=${SYS_CLK_KHZ})
target_compile_definitions(${PROJECT_NAME} PUBLIC DEFAULT_PERI_CLK_KHZ=${PERI_CLK_KHZ})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_DB_BASE=${LCD_PIN_DB_BASE})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_DB_COUNT=${LCD_PIN_DB_COUNT})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_CS=${LCD_PIN_CS})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_WR=${LCD_PIN_WR})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_RS=${LCD_PIN_RS})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_RST=${LCD_PIN_RST})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_PIN_BL=${LCD_PIN_BL})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_ROTATION=${LCD_ROTATION})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_HOR_RES=${LCD_HOR_RES})
target_compile_definitions(${PROJECT_NAME} PUBLIC LCD_VER_RES=${LCD_VER_RES})
target_compile_definitions(${PROJECT_NAME} PUBLIC DISP_OVER_PIO=${DISP_OVER_PIO})
target_compile_definitions(${PROJECT_NAME} PUBLIC MY_DISP_BUF_SIZE=${MY_DISP_BUF_SIZE})

# Note: If you are using a NOR flash like "w25q16". Just keep the following content.
# The maximum speed of "w25q16" is 133MHz, However, the clock speed of XIP QSPI is divided from "sys_clk".
# So, when your "sys_clk" is greater than 266MHz and default PICO_FLASH_SPI_CLKDIV=2, It will exceed the
# maximum speed, because PICO_FLASH_SPI_CLKDIV must be even, So 4 is good for most purpose, Otherwise,
# nothing should be done. These things will only useful when you overclocking the rp2040.
if(${PICO_BOARD} STREQUAL "pico" OR ${PICO_PLATFORM} STREQUAL "rp2040")
    message(FATAL_ERROR "This demo does not support the pico board.")
elseif(${PICO_BOARD} STREQUAL  "pico2" OR ${PICO_PLATFORM} STREQUAL  "rp2350")
	set(PICO_FLASH_SPI_CLKDIV 3)
endif()

math(EXPR FLASH_CLK_KHZ "${SYS_CLK_KHZ} / ${PICO_FLASH_SPI_CLKDIV}")
math(EXPR FLASH_CLK_MHZ "${FLASH_CLK_KHZ} / 1000")
math(EXPR SYS_CLK_MHZ "${SYS_CLK_KHZ} / 1000")
math(EXPR DISP_BUF_SIZE "${MY_DISP_BUF_SIZE} * 2") # rgb565 cost 2 bytes
message(WARNING "
    CPU speed   : ${SYS_CLK_MHZ} MHz
    Flash speed : ${FLASH_CLK_MHZ} MHz
    LCD_X       : ${LCD_HOR_RES}
    LCD_Y       : ${LCD_VER_RES}
    LCD rotation: ${LCD_ROTATION}   //0: normal, 1: 90 degree, 2: 180 degree, 3: 270 degree
    Buffer size : ${DISP_BUF_SIZE} bytes (LVGL Draw Buffer)
")
target_compile_definitions(bs2_default PRIVATE PICO_FLASH_SPI_CLKDIV=${PICO_FLASH_SPI_CLKDIV})
target_compile_definitions(${PROJECT_NAME} PRIVATE FLASH_CLK_KHZ=${FLASH_CLK_KHZ})

pico_enable_stdio_usb(${PROJECT_NAME} 0)    # 0: disable, 1: enable
pico_enable_stdio_uart(${PROJECT_NAME} 1)   # 0: disable, 1: enable

pico_add_extra_outputs(${PROJECT_NAME}) # uf2 and more files

# show target size info
add_custom_target(
    print-memory-usage ALL
    COMMAND arm-none-eabi-size -G ${CMAKE_PROJECT_NAME}.elf
    DEPENDS ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.elf
    COMMENT "Print target size info"
    DEPENDS ${PROJECT_NAME}
)

# add a firmware flash target
if(${PICO_BOARD} STREQUAL "pico" OR ${PICO_PLATFORM} STREQUAL "rp2040")
    message(FATAL_ERROR "This demo does not support the pico board.")
elseif(${PICO_BOARD} STREQUAL  "pico2" OR ${PICO_PLATFORM} STREQUAL  "rp2350")
	add_custom_target(
	    flash
	    COMMAND openocd -f interface/cmsis-dap.cfg -c "adapter speed 10000"
		            -f target/rp2350.cfg -s tcl -c "program ${CMAKE_PROJECT_NAME}.elf verify reset exit"
	    DEPENDS ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.elf
	    COMMENT "Flashing firmware using CMSIS-DAP Debugger..."
	    DEPENDS ${PROJECT_NAME}
	)
endif()
