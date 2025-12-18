cmake_minimum_required(VERSION 3.20)

get_filename_component(PROJECT_NAME ${CMAKE_SOURCE_DIR} NAME)
project(${PROJECT_NAME} C ASM)

set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR cortex-m4)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)

set(TOOLCHAIN_PREFIX arm-none-eabi)
set(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}-gcc)
set(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}-gcc)
set(CMAKE_OBJCOPY      ${TOOLCHAIN_PREFIX}-objcopy)
set(CMAKE_SIZE         ${TOOLCHAIN_PREFIX}-size)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug)
endif()

set(SRC_DIR ${CMAKE_SOURCE_DIR}/src)
set(INC_DIR ${CMAKE_SOURCE_DIR}/inc)

file(GLOB_RECURSE C_SOURCES ${SRC_DIR}/*.c)
set(SOURCES ${C_SOURCES} ${CMAKE_SOURCE_DIR}/startup_stm32f411xe.s)

set(MCU_FLAGS -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard)

if(CMAKE_BUILD_TYPE STREQUAL "Release")
    set(COMP_FLAGS ${MCU_FLAGS} -O2 -Wall -ffunction-sections -fdata-sections)
    add_compile_definitions(STM32 STM32F4 STM32F411xE USE_STDPERIPH_DRIVER)
else()
    set(COMP_FLAGS ${MCU_FLAGS} -O0 -g3 -Wall -ffunction-sections -fdata-sections)
    add_compile_definitions(STM32 STM32F4 STM32F411xE USE_STDPERIPH_DRIVER DEBUG)
endif()
add_compile_options(${COMP_FLAGS})

add_executable(${PROJECT_NAME}.elf ${SOURCES})
target_include_directories(${PROJECT_NAME}.elf PRIVATE ${INC_DIR})

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/STM32F411CEUX_FLASH.ld)
target_link_options(${PROJECT_NAME}.elf PRIVATE ${MCU_FLAGS} -T${LINKER_SCRIPT} -Wl,--gc-sections -Wl,-Map=${CMAKE_BINARY_DIR}/${PROJECT_NAME}.map)

add_custom_command(
    TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
)

add_custom_target(release
    COMMAND ${CMAKE_COMMAND} -DCMAKE_BUILD_TYPE=Release -S ${CMAKE_SOURCE_DIR} -B ${CMAKE_BINARY_DIR}/release
    COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}/release --target ${PROJECT_NAME}.elf
)

add_custom_target(flash
    COMMAND st-flash --reset write ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.bin 0x08000000
)

add_custom_target(flash-release
    COMMAND st-flash --reset write ${CMAKE_BINARY_DIR}/release/${PROJECT_NAME}.bin 0x08000000
)

add_custom_target(size
    COMMAND ${CMAKE_SIZE} ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.elf
)

add_custom_target(size-release
    COMMAND ${CMAKE_SIZE} ${CMAKE_BINARY_DIR}/release/${PROJECT_NAME}.elf
)

add_custom_target(clean-all
    COMMAND ${CMAKE_COMMAND} -E rm -rf ${CMAKE_BINARY_DIR}/*
)

