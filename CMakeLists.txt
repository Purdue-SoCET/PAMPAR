set(CMAKE_C_COMPILER riscv64-unknown-elf-gcc)
set(CMAKE_CXX_COMPILER riscv64-unknown-elf-g++)

cmake_minimum_required(VERSION 3.13)
project(aftx07-pampar)

enable_language(ASM)
include(CMakePrintHelpers)

set(COMMON_COMPILE_OPTIONS
    -march=rv32imc_zicsr_zifencei
    -mabi=ilp32
    -mcmodel=medany
    -static
    -nostdlib
    -nostartfiles
    -ffreestanding
    -ffunction-sections
    -fdata-sections
    -Oz
    -Wall
    -Wextra
    -pedantic
)

# usage: -DDEBUG=true|false
if (DEBUG)
    set(COMMON_COMPILE_OPTIONS
        ${COMMON_COMPILE_OPTIONS}
        -D DEBUG
    )
endif()

set(LINK_OPTIONS
    "-T${CMAKE_CURRENT_SOURCE_DIR}/link.ld"
    -Wl,--gc-sections
    -Wl,--warn-constructors
)

add_compile_options(${COMMON_COMPILE_OPTIONS})
add_link_options(${COMMON_COMPILE_OPTIONS} ${LINK_OPTIONS})

file(GLOB LIB_SOURCE AFTx07/sw-tests/support/*.c AFTx07/sw-tests/support/*.S)
file(GLOB LIB_HEADER AFTx07/sw-tests/support/*.h *.h)
#add_library(FemtoKernel STATIC ${LIB_SOURCE} ${LIB_HEADER})

cmake_print_variables(LIB_SOURCE)


set(APPS
    DFT
    DJ
    DP
    GL
    GS
    HA
    JA
    MM
    NI
    OE
    PI
    SH
    TR)
file(GLOB TESTS "Apps/*/sequential*" "Apps/*/pthread*")
foreach( filename ${TESTS} )
    string(REGEX REPLACE ".*Apps/(.*)/((sequential|pthread))\..*" "\\1_\\2" targetname ${filename})
    cmake_print_variables(targetname)
    add_executable(${targetname} ${filename} ${LIB_SOURCE} ${LIB_HEADER})
    target_include_directories(${targetname} PUBLIC support)
    set_target_properties(
        ${targetname}
        PROPERTIES
            SUFFIX ".elf"
    )

    add_custom_command(
        TARGET ${targetname}
        POST_BUILD
        COMMAND riscv64-unknown-elf-objcopy -O binary ${targetname}.elf ${targetname}.bin
        COMMAND riscv64-unknown-elf-objdump -d ${targetname}.elf > ${targetname}.dump
        COMMENT "Generating binary file and objdump"
    )

    set_target_properties(${targetname} PROPERTIES ADDITIONAL_CLEAN_FILES
            "${targetname}.bin;${targetname}.dump")
endforeach()
