cmake_minimum_required(VERSION 3.16)
project(Tinos3cKernel C ASM)

set(CMAKE_C_FLAGS "-m64 -std=gnu11 -ffreestanding -Wall -Wextra -nostdlib -I kernel -I $(UACPI_DIR)/include -fno-stack-protector -Wno-unused-parameter -fno-stack-check -fno-lto -mno-mmx -mno-80387 -mno-sse -mno-sse2 -mno-red-zone -fno-pic -mcmodel=kernel")
set(CMAKE_EXE_LINKER_FLAGS "-ffreestanding -nostdlib -lgcc -T ${CMAKE_SOURCE_DIR}/kernel/linker.ld")

file(GLOB_RECURSE KERNEL_SOURCES
        CONFIGURE_DEPENDS
        "${CMAKE_SOURCE_DIR}/kernel/*.c"
        "${CMAKE_SOURCE_DIR}/kernel/*.S"
)


include(external/uacpi/uacpi.cmake)

add_custom_target(iso ALL
        COMMAND grub-mkrescue -o ${CMAKE_BINARY_DIR}/tinoscmake.iso ${CMAKE_BINARY_DIR}/bin
        DEPENDS Tinos3cKernel
)

set_target_properties(Tinos3cKernel PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}
        LINKER_LANGUAGE C
)

target_sources(
        Tinos3cKernel
        PRIVATE
        ${UACPI_SOURCES}
)

target_include_directories(
        Tinos3cKernel
        PRIVATE
        ${UACPI_INCLUDES}
)

add_executable(kernel.elf
        ${KERNEL_SOURCES}
        ${UACPI_SOURCES}
)