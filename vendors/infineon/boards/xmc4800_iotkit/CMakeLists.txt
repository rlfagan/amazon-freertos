# Set variables.
set(xmc4800_dir "${AFR_VENDORS_DIR}/infineon/boards/xmc4800_iotkit")
set(xmc4800_ports_dir "${xmc4800_dir}/ports")
set(xmclib_dir "${AFR_VENDORS_DIR}/infineon/XMCLib/2.1.20")

if(AFR_IS_TESTING)
    set(xmc4800_aws_dir "${xmc4800_dir}/aws_tests")
else()
    set(xmc4800_aws_dir "${xmc4800_dir}/aws_demos")
endif()

#Tool chain specific check
if(NOT ${AFR_TOOLCHAIN} STREQUAL "arm-gcc")
   if(NOT AFR_METADATA_MODE)
       message( FATAL_ERROR "The toolchain is not supported." )
   endif()
endif()

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS Console metadata
# -------------------------------------------------------------------------------------------------

afr_set_board_metadata(ID "Infineon-XMC4800-IoT-Kit")
afr_set_board_metadata(DISPLAY_NAME "XMC4800 IoT Connectivity Kit")
afr_set_board_metadata(DESCRIPTION "Development kit for ARM� Cortex�-M4 based XMC4800 MCU")
afr_set_board_metadata(VENDOR_NAME "Infineon")
afr_set_board_metadata(FAMILY_NAME "XMC4x00")
afr_set_board_metadata(CODE_SIGNER "null")
afr_set_board_metadata(DATA_RAM_MEMORY "353KB")
afr_set_board_metadata(PROGRAM_MEMORY "2MB")
afr_set_board_metadata(SUPPORTED_IDE "DAVE")
afr_set_board_metadata(RECOMMENDED_IDE "DAVE")
afr_set_board_metadata(IDE_DAVE_NAME "DAVE")
afr_set_board_metadata(IDE_DAVE_COMPILER "GCC")

afr_set_board_metadata(IDE_DAVE_PROJECT_LOCATION "${AFR_ROOT_DIR}/vendors/infineon/boards/xmc4800_iotkit/aws_demos/dave4")
afr_set_board_metadata(AWS_DEMOS_CONFIG_FILES_LOCATION "${AFR_ROOT_DIR}/vendors/infineon/boards/xmc4800_iotkit/aws_demos/common/config_files")

# -------------------------------------------------------------------------------------------------
# Compiler settings
# -------------------------------------------------------------------------------------------------
afr_mcu_port(compiler)


#Set defined symbols for compiler and assembler 
set( defined_symbols
     XMC4800_F100x2048
)

# Compiler defined symbols.
target_compile_definitions(
    AFR::compiler::mcu_port
    INTERFACE
    ${defined_symbols}
)

#Set Compiler Debug options
set( compiler_debug_options "-g" "-gdwarf-2")

#Set compiler warnings level
set( compiler_warnings "-Wall" )

#Set compiler optimization level
set( compiler_optimization_options "-O2" "-ffunction-sections" "-fdata-sections")

#Set ARM GCC options
set( compiler_arm_options "-mfloat-abi=softfp" "-mcpu=cortex-m4" "-mfpu=fpv4-sp-d16" "-mthumb")

#Set other compiler options and flags
set( compiler_misc_options
     "-std=gnu99"
     "-c"
     "-fmessage-length=0"
)

#All compiler options
set( compiler_flags
     ${compiler_debug_options}
     ${compiler_warnings}
     ${compiler_optimization_options}
     ${compiler_arm_options}
     ${compiler_misc_options}
     "-pipe" )
target_compile_options(
    AFR::compiler::mcu_port
    INTERFACE
        $<$<COMPILE_LANGUAGE:C>:${compiler_flags}>
)


#All assembler options
set( assembler_flags
     "-x"
     "assembler-with-cpp"
     ${compiler_debug_options}
     ${compiler_warnings}
     ${compiler_optimization_options}
     ${compiler_arm_options}
     ${compiler_misc_options}
)
target_compile_options(
    AFR::compiler::mcu_port
    INTERFACE
        $<$<COMPILE_LANGUAGE:ASM>:${assembler_flags}>
)


#Linker script path
set( linker_script "${xmc4800_aws_dir}/dave4/XMC4800x2048.ld" )

#All linker options
set( linker_flags
     "-T" ${linker_script}
     "-nostartfiles"
     "-Xlinker"
     "--gc-sections"
     "-specs=nosys.specs"
     "-Wl,-Map,aws_demos.map"
     ${compiler_arm_options}
     ${compiler_debug_options}
)

target_link_options(
    AFR::compiler::mcu_port
    INTERFACE
        ${linker_flags}
)

#Linker libraries
set(linker_libs
    "m"
)
target_link_libraries(
    AFR::compiler::mcu_port
    INTERFACE
        ${linker_libs}
)

# -------------------------------------------------------------------------------------------------
# Kernel and Amazon FreeRTOS portable layers
# -------------------------------------------------------------------------------------------------

# Kernel
afr_mcu_port(kernel)

#Driver source files
afr_glob_src(driver_src DIRECTORY "${xmclib_dir}/drivers/src")

#Source files specific to compiler
set(compiler_src
    "${AFR_KERNEL_DIR}/portable/GCC/ARM_CM4F/port.c"
    "${AFR_KERNEL_DIR}/portable/GCC/ARM_CM4F/portmacro.h"
    "${xmclib_dir}/CMSIS/Infineon/XMC4800_series/Source/system_XMC4800.c"
    "${xmclib_dir}/CMSIS/Infineon/XMC4800_series/Source/GCC/startup_XMC4800.S"
)

target_sources(
    AFR::kernel::mcu_port
    INTERFACE
       "${AFR_KERNEL_DIR}/portable/MemMang/heap_4.c"
        ${compiler_src}
        ${driver_src}
)

#Include directories specific to compiler
set(compiler_includes
    "${AFR_KERNEL_DIR}/portable/GCC/ARM_CM4F"
)
set( board_includes
     "${xmclib_dir}/CMSIS/Include"
     "${xmclib_dir}/drivers/inc"
     "${xmclib_dir}/third_party/ESP_AT_Lib/src/include"
     "${xmclib_dir}/CMSIS/Infineon/XMC4800_series/Include"
     "${xmc4800_aws_dir}/common/application_code/infineon_code"

)
set(kernel_includes
    ${board_includes}
    ${compiler_includes}
    "${AFR_KERNEL_DIR}/include"
    "$<IF:${AFR_IS_TESTING},${AFR_TEST_DIR},${AFR_DEMOS_DIR}>/include"
    "${AFR_3RDPARTY_DIR}/mbedtls/include"
    "${xmc4800_aws_dir}/common/config_files"
)

target_include_directories(
    AFR::kernel::mcu_port
    INTERFACE $<$<NOT:$<COMPILE_LANGUAGE:ASM>>:${kernel_includes}>
)

# WiFi
afr_mcu_port(wifi)
target_sources(
    AFR::wifi::mcu_port
    INTERFACE "${xmc4800_ports_dir}/wifi/aws_wifi.c"
)

# PKCS11
afr_mcu_port(pkcs11)
target_sources(
    AFR::pkcs11::mcu_port
    INTERFACE
        "${xmc4800_ports_dir}/pkcs11/aws_pkcs11_pal.c"
)

target_link_libraries(
    AFR::pkcs11::mcu_port
    INTERFACE
        3rdparty::mbedtls
        AFR::pkcs11_mbedtls
        AFR::crypto
)

# Secure sockets
afr_mcu_port(secure_sockets)
target_link_libraries(
    AFR::secure_sockets::mcu_port
    INTERFACE
        AFR::tls
        AFR::wifi
        AFR::pkcs11
)

target_sources(
    AFR::secure_sockets::mcu_port
    INTERFACE "${xmc4800_ports_dir}/secure_sockets/aws_secure_sockets.c"
)

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS demos and tests
# -------------------------------------------------------------------------------------------------
set(CMAKE_EXECUTABLE_SUFFIX ".elf")

if(AFR_IS_TESTING)
    set(exe_target aws_tests)
else()
    set(exe_target aws_demos)
endif()


afr_glob_src(esp_at_lib RECURSE DIRECTORY "${xmclib_dir}/third_party/ESP_AT_Lib/src")
afr_glob_src(board_src DIRECTORY "${xmc4800_aws_dir}/common/application_code/infineon_code")
afr_glob_src(newlib DIRECTORY "${xmclib_dir}/Newlib")
afr_glob_src(config_src DIRECTORY "${xmc4800_aws_dir}/common/config_files")

# TODO, remove network manager src.
afr_glob_src(network_manager_src DIRECTORY "${AFR_DEMOS_DIR}/network_manager")

add_executable( 
    ${exe_target}
    ${board_src}
    ${config_src}
    ${newlib}
    ${esp_at_lib}
    "${xmc4800_aws_dir}/common/application_code/main.c"
    $<$<NOT:${AFR_IS_TESTING}>:${network_manager_src}>
)

target_link_libraries(
    ${exe_target}
    PRIVATE
        AFR::wifi
        AFR::utils
)

if(AFR_METADATA_MODE)
    return()
endif()

find_program(gcc_objectcopy arm-none-eabi-objcopy)
find_program(gcc_size arm-none-eabi-size)

if(NOT gcc_objectcopy )
    message(FATAL_ERROR "Cannot find arm-none-eabi-objcopy.")
endif()

set(output_file "$<TARGET_FILE_DIR:${exe_target}>/${exe_target}.hex")

add_custom_command(
    TARGET ${exe_target} POST_BUILD
    COMMAND "${gcc_objectcopy}" -O ihex "$<TARGET_FILE:${exe_target}>" "${output_file}"
    COMMAND "${gcc_size}" "$<TARGET_FILE:${exe_target}>"
)

add_custom_command(
    TARGET ${exe_target} POST_BUILD
    COMMAND "${CMAKE_COMMAND}" -E copy "${output_file}" "${CMAKE_BINARY_DIR}"
)