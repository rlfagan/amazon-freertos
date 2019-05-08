set(afr_ports_dir "${CMAKE_CURRENT_LIST_DIR}/ports")
set(board_demos_dir "${CMAKE_CURRENT_LIST_DIR}/aws_demos/common")
set(board_tests_dir "${CMAKE_CURRENT_LIST_DIR}/aws_tests/common")
if(AFR_IS_TESTING)
    set(board_dir "${board_tests_dir}")
else()
    set(board_dir "${board_demos_dir}")
endif()

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS Console metadata
# -------------------------------------------------------------------------------------------------

# -------------------------------------------------------------------------------------------------
# Compiler settings
# -------------------------------------------------------------------------------------------------
afr_mcu_port(compiler)

target_compile_definitions(
    AFR::compiler::mcu_port
    INTERFACE
        __free_rtos__
        _CONSOLE
        _CRT_SECURE_NO_WARNINGS
)

target_compile_options(
    AFR::compiler::mcu_port
    INTERFACE "/MP" "/W4" "/wd4210" "/wd4127" "/wd4244" "/wd4310"
)

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS portable layers
# -------------------------------------------------------------------------------------------------
# Normally the portable layer for kernel should be vendor's driver code.
afr_mcu_port(kernel)
target_sources(
    AFR::kernel::mcu_port
    INTERFACE
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW/port.c"
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW/portmacro.h"
        "${AFR_KERNEL_DIR}/portable/MemMang/heap_4.c"
)
target_include_directories(
    AFR::kernel::mcu_port
    INTERFACE
        "${AFR_KERNEL_DIR}/portable/MSVC-MingW"
        "${board_dir}/config_files"
        "${board_dir}/application_code"
        # Need aws_clientcredential.h
        "$<IF:${AFR_IS_TESTING},${AFR_TEST_DIR},${AFR_DEMOS_DIR}>/include"
)
target_link_libraries(
    AFR::kernel::mcu_port
    INTERFACE
        3rdparty::tracealyzer
)

# POSIX
afr_mcu_port(posix)
target_sources(
    AFR::posix::mcu_port
    INTERFACE "${afr_ports_dir}/posix/FreeRTOS_POSIX_portable.h"
)
target_include_directories(
    AFR::posix::mcu_port
    INTERFACE "${afr_ports_dir}/posix"
)
target_link_libraries(
    AFR::posix::mcu_port
    INTERFACE AFR::freertos_plus_posix
)

# PKCS11
afr_mcu_port(pkcs11)
target_sources(
    AFR::pkcs11::mcu_port
    INTERFACE
        "${afr_ports_dir}/pkcs11/aws_pkcs11_pal.c"
        "${AFR_MODULES_PORTS_DIR}/pkcs11/mbedtls/threading_alt.h"
)
target_link_libraries(
    AFR::pkcs11::mcu_port
    INTERFACE
        AFR::pkcs11_mbedtls
        3rdparty::mbedtls
)

# FreeRTOS Plus TCP
afr_mcu_port(freertos_plus_tcp)
target_sources(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE
        "${AFR_MODULES_STANDARD_DIR}/freertos_plus_tcp/source/portable/BufferManagement/BufferAllocation_2.c"
        "${AFR_MODULES_STANDARD_DIR}/freertos_plus_tcp/source/portable/NetworkInterface/WinPCap/NetworkInterface.c"
)
target_include_directories(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE
        "${AFR_MODULES_STANDARD_DIR}/freertos_plus_tcp/source/portable/Compiler/MSVC"
)
target_link_libraries(
    AFR::freertos_plus_tcp::mcu_port
    INTERFACE 3rdparty::winpcap
)

# Secure sockets
afr_mcu_port(secure_sockets)
target_link_libraries(
    AFR::secure_sockets::mcu_port
    INTERFACE AFR::secure_sockets_freertos_tcp
)

# OTA
afr_mcu_port(ota)
target_sources(
    AFR::ota::mcu_port
    INTERFACE "${afr_ports_dir}/ota/aws_ota_pal.c"
)
target_link_libraries(
    AFR::ota::mcu_port
    INTERFACE AFR::crypto
)

# -------------------------------------------------------------------------------------------------
# Amazon FreeRTOS demos and tests
# -------------------------------------------------------------------------------------------------
# TODO, remove network manager src.
afr_glob_src(network_manager_src DIRECTORY "${AFR_DEMOS_DIR}/network_manager")

if(AFR_IS_TESTING)
    set(exe_target aws_tests)
else()
    set(exe_target aws_demos)
    set(extra_src ${network_manager_src})
endif()

add_executable(
    ${exe_target}
    "${board_dir}/application_code/main.c"
    "${board_demos_dir}/application_code/aws_demo_logging.c"
    "${board_demos_dir}/application_code/aws_demo_logging.h"
    "${board_demos_dir}/application_code/aws_entropy_hardware_poll.c"
    "${board_demos_dir}/application_code/aws_run-time-stats-utils.c"
    # Need to add cborpretty.c
    "${AFR_3RDPARTY_DIR}/tinycbor/cborpretty.c"
    ${extra_src}
)
target_include_directories(
    ${exe_target}
    PRIVATE
        "${board_demos_dir}/application_code"
)
target_link_libraries(
    ${exe_target}
    PRIVATE
        AFR::freertos_plus_tcp
        AFR::utils
)