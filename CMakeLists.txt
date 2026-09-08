cmake_minimum_required(VERSION 3.16)

project(Tinos3cKernel C)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ------------------------------------------------------------
# Tools
# ------------------------------------------------------------

find_program(NASM_EXECUTABLE nasm REQUIRED)
find_program(QEMU_EXECUTABLE qemu-system-x86_64 REQUIRED)
find_program(GRUB_FILE_EXECUTABLE grub-file REQUIRED)
find_program(GRUB_MKRESCUE_EXECUTABLE grub-mkrescue REQUIRED)

# ------------------------------------------------------------
# uACPI
# ------------------------------------------------------------

include(external/uacpi/uacpi.cmake)

# ------------------------------------------------------------
# Source files
# ------------------------------------------------------------

file(GLOB_RECURSE KERNEL_C_SOURCES
CONFIGURE_DEPENDS
"${CMAKE_SOURCE_DIR}/kernel/*.c"
)

file(GLOB_RECURSE KERNEL_ASM_SOURCES
CONFIGURE_DEPENDS
"${CMAKE_SOURCE_DIR}/kernel/*.S"
)

# ------------------------------------------------------------
# NASM assembly
# ------------------------------------------------------------

set(KERNEL_ASM_OBJECTS)

foreach(ASM_SOURCE ${KERNEL_ASM_SOURCES})

file(RELATIVE_PATH ASM_RELATIVE
"${CMAKE_SOURCE_DIR}"
"${ASM_SOURCE}"
)

string(REPLACE "/" "_" ASM_NAME "${ASM_RELATIVE}")

set(ASM_OBJECT
"${CMAKE_BINARY_DIR}/asm/${ASM_NAME}.o"
)

add_custom_command(
OUTPUT "${ASM_OBJECT}"

COMMAND ${CMAKE_COMMAND} -E make_directory
"${CMAKE_BINARY_DIR}/asm"

COMMAND ${NASM_EXECUTABLE}
-f elf64
-g
-F dwarf
-o "${ASM_OBJECT}"
"${ASM_SOURCE}"

DEPENDS "${ASM_SOURCE}"

VERBATIM
)

list(APPEND KERNEL_ASM_OBJECTS "${ASM_OBJECT}")

endforeach()

# ------------------------------------------------------------
# Kernel
# ------------------------------------------------------------

add_executable(Tinos3cKernel
${KERNEL_C_SOURCES}
${UACPI_SOURCES}
${KERNEL_ASM_OBJECTS}
)

set_target_properties(Tinos3cKernel PROPERTIES
OUTPUT_NAME "kernel.elf"
RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
LINKER_LANGUAGE C
)

# ------------------------------------------------------------
# Compiler flags
# ------------------------------------------------------------

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

# ------------------------------------------------------------
# Include directories
# ------------------------------------------------------------

target_include_directories(Tinos3cKernel PRIVATE
"${CMAKE_SOURCE_DIR}/kernel"
${UACPI_INCLUDES}
)

# ------------------------------------------------------------
# Linker
# ------------------------------------------------------------

target_link_options(Tinos3cKernel PRIVATE
        -nostdlib
        -Wl,-m,elf_x86_64
        "-Wl,-T,${CMAKE_SOURCE_DIR}/kernel/linker.ld"
        -Wl,-z,noexecstack
)

# ------------------------------------------------------------
# ISO
# ------------------------------------------------------------

set(ISO_ROOT
"${CMAKE_BINARY_DIR}/iso_root"
)

set(ISO_FILE
"${CMAKE_BINARY_DIR}/tinoscmake.iso"
)

add_custom_command(
OUTPUT "${ISO_FILE}"

COMMAND ${CMAKE_COMMAND} -E make_directory
"${ISO_ROOT}/boot/grub"

COMMAND ${CMAKE_COMMAND} -E copy
"$<TARGET_FILE:Tinos3cKernel>"
"${ISO_ROOT}/boot/kernel.elf"

COMMAND ${CMAKE_COMMAND} -E copy
"${CMAKE_SOURCE_DIR}/boot/grub.cfg"
"${ISO_ROOT}/boot/grub/grub.cfg"

COMMAND ${GRUB_FILE_EXECUTABLE}
--is-x86-multiboot
"$<TARGET_FILE:Tinos3cKernel>"

COMMAND ${GRUB_MKRESCUE_EXECUTABLE}
-o "${ISO_FILE}"
"${ISO_ROOT}"

DEPENDS Tinos3cKernel

VERBATIM
)

add_custom_target(iso ALL
DEPENDS "${ISO_FILE}"
)

# ------------------------------------------------------------
# Run in QEMU
# ------------------------------------------------------------

add_custom_target(run
COMMAND ${QEMU_EXECUTABLE}
-m 192M
-debugcon stdio
-cdrom "${ISO_FILE}"
-boot d

DEPENDS iso

USES_TERMINAL
)

# ------------------------------------------------------------
# Run QEMU with GDB
# ------------------------------------------------------------

add_custom_target(run-gdb
COMMAND ${QEMU_EXECUTABLE}
-m 192M
-debugcon stdio
-cdrom "${ISO_FILE}"
-boot d
-S
-s

DEPENDS iso

USES_TERMINAL
)