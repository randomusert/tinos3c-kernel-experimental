cmake_minimum_required(VERSION 3.16)

project(Tinos3cKernel C ASM)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

file(GLOB_RECURSE KERNEL_SOURCES
        CONFIGURE_DEPENDS
        "${CMAKE_SOURCE_DIR}/kernel/*.c"
        "${CMAKE_SOURCE_DIR}/kernel/*.S"
)

include(external/uacpi/uacpi.cmake)

add_executable(Tinos3cKernel
        ${KERNEL_SOURCES}
        ${UACPI_SOURCES}
)

set_target_properties(Tinos3cKernel PROPERTIES
        OUTPUT_NAME "kernel.elf"
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
        LINKER_LANGUAGE C
)

target_compile_options(Tinos3cKernel PRIVATE
        -m64
        -std=gnu11
        -ffreestanding
        -Wall
        -Wextra
        -nostdlib
        -fno-stack-protector
        -Wno-unused-parameter
        -fno-stack-check
        -fno-lto
        -mno-mmx
        -mno-80387
        -mno-sse
        -mno-sse2
        -mno-red-zone
        -fno-pic
        -mcmodel=kernel
)

target_include_directories(Tinos3cKernel PRIVATE
        "${CMAKE_SOURCE_DIR}/kernel"
        ${UACPI_INCLUDES}
)

target_link_options(Tinos3cKernel PRIVATE
        -nostdlib
        -T "${CMAKE_SOURCE_DIR}/kernel/linker.ld"
        -Wl,-z,noexecstack
)

set(ISO_ROOT "${CMAKE_BINARY_DIR}/iso_root")

add_custom_command(
        OUTPUT "${CMAKE_BINARY_DIR}/tinoscmake.iso"

        COMMAND ${CMAKE_COMMAND} -E make_directory
        "${ISO_ROOT}/boot/grub"

        COMMAND ${CMAKE_COMMAND} -E copy
        "$<TARGET_FILE:Tinos3cKernel>"
        "${ISO_ROOT}/boot/kernel.elf"

        COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_SOURCE_DIR}/boot/grub.cfg"
        "${ISO_ROOT}/boot/grub/grub.cfg"

        COMMAND grub-file --is-x86-multiboot
        "$<TARGET_FILE:Tinos3cKernel>"

        COMMAND grub-mkrescue
        -o "${CMAKE_BINARY_DIR}/tinoscmake.iso"
        "${ISO_ROOT}"

        DEPENDS Tinos3cKernel
)

add_custom_target(iso ALL
        DEPENDS "${CMAKE_BINARY_DIR}/tinoscmake.iso"
)