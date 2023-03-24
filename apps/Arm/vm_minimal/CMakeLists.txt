#
# Copyright 2018, Data61, CSIRO (ABN 41 687 119 230)
#
# SPDX-License-Identifier: BSD-2-Clause
#

cmake_minimum_required(VERSION 3.8.2)

project(camkes-arm-tk1-vm C)

include(${CAMKES_ARM_VM_HELPERS_PATH})

# On some platforms, there is no dedicated rootfs, because it's part of the
# Linux kernel image.
set(rootfs_file "")

# Create our CPP Flags based on ARM VM config variables
if("${KernelARMPlatform}" STREQUAL "tk1")
    set(cpp_flags "-DKERNELARMPLATFORM_TK1")

    # Different device tree if compiling with Tk1Insecure
    if(Tk1Insecure)
        set(device_tree_src "${CAMKES_ARM_LINUX_DIR}/linux-tk1-nonsecured.dts")
    else()
        set(device_tree_src "${CAMKES_ARM_LINUX_DIR}/linux-tk1-secure.dts")
    endif()

    # kernel image contains rootfs
    if(VmTk1InitrdRootfs)
        set(linux_image "${CAMKES_ARM_LINUX_DIR}/linux-tk1-initrd")
    else()
        set(linux_image "${CAMKES_ARM_LINUX_DIR}/linux-tk1-debian")
    endif()

    set(output_dtb_location "${CMAKE_CURRENT_BINARY_DIR}/linux/linux-dtb")
    add_custom_command(
        OUTPUT "${output_dtb_location}"
        COMMAND bash -c "which dtc && dtc -I dts -O dtb ${device_tree_src} > ${output_dtb_location}"
        VERBATIM
        DEPENDS ${device_tree_src}
    )
    # Create custom target for setting the dtb
    add_custom_target(dtb_gen_target DEPENDS "${linux-dtb}")

elseif("${KernelARMPlatform}" STREQUAL "tx1")
    set(cpp_flags "-DKERNELARMPLATFORM_TX1")

    # kernel image contains rootfs
    set(linux_image "${CAMKES_VM_IMAGES_DIR}/tx1/linux")

    # There is no need to generate or modify a DTB, so we just have a dummy
    # target here. The target is depends on the (existing) DTB file, so we get a
    # nice error here in case it is missing.
    set(output_dtb_location "${CAMKES_VM_IMAGES_DIR}/tx1/linux-dtb")
    add_custom_target(dtb_gen_target DEPENDS "${output_dtb_location}")

elseif("${KernelARMPlatform}" STREQUAL "exynos5422")
    find_package(camkes-vm-linux REQUIRED)
    include(${CAMKES_VM_LINUX_HELPERS_PATH})
    set(cpp_flags "-DKERNELARMPLATFORM_EXYNOS5422")

    set(linux_image "${CAMKES_VM_IMAGES_DIR}/exynos5422/linux")
    # Generate overlayed rootfs
    set(rootfs_file "${CAMKES_VM_IMAGES_DIR}/exynos5422/rootfs.cpio.gz")
    # Updated dtb based on initrd
    UpdateDtbFromInitrd(
        "${CAMKES_VM_IMAGES_DIR}/exynos5422/linux-dtb"
        ${rootfs_file}
        "0x4d700000"
        dtb_gen_target
        output_dtb_location
    )

elseif("${KernelARMPlatform}" STREQUAL "tx2")
    find_package(camkes-vm-linux REQUIRED)
    include(${CAMKES_VM_LINUX_HELPERS_PATH})
    set(cpp_flags "-DKERNELARMPLATFORM_TX2")

    set(linux_image "${CAMKES_VM_IMAGES_DIR}/tx2/linux")
    set(rootfs_file "${CAMKES_VM_IMAGES_DIR}/tx2/rootfs.cpio.gz")
    # Updated dtb based on initrd
    UpdateDtbFromInitrd(
        "${CAMKES_VM_IMAGES_DIR}/tx2/linux-dtb"
        ${rootfs_file}
        "0xf7000000"
        dtb_gen_target
        output_dtb_location
    )

elseif("${KernelARMPlatform}" STREQUAL "qemu-arm-virt")
    find_package(camkes-vm-linux REQUIRED)
    include(${CAMKES_VM_LINUX_HELPERS_PATH})
    set(cpp_flags "-DKERNELARMPLATFORM_QEMU-ARM-VIRT")

    set(linux_image "${CAMKES_VM_IMAGES_DIR}/qemu-arm-virt/linux")
    set(rootfs_file "${CAMKES_VM_IMAGES_DIR}/qemu-arm-virt/rootfs.cpio.gz")
    # Updated dtb based on initrd
    UpdateDtbFromInitrd(
        "${CAMKES_VM_IMAGES_DIR}/qemu-arm-virt/linux-dtb"
        ${rootfs_file}
        "0x4d700000"
        dtb_gen_target
        output_dtb_location
    )
    include(simulation)
    set(SIMULATION ON CACHE BOOL "Generate simulation script to run qemu with the proper arguments")
    if(SIMULATION)
        GenerateSimulateScript()
    endif()

elseif("${KernelARMPlatform}" STREQUAL "odroidc2")
    find_package(camkes-vm-linux REQUIRED)
    include(${CAMKES_VM_LINUX_HELPERS_PATH})
    set(cpp_flags "-DKERNELARMPLATFORM_ODROIDC2")

    set(
        CAMKES_ROOT_DTB_FILE_PATH "${CAMKES_VM_IMAGES_DIR}/odroidc2/camkes-linux-dtb"
        CACHE STRING ""
    )
    set(linux_image "${CAMKES_VM_IMAGES_DIR}/odroidc2/linux")
    set(rootfs_file "${CAMKES_VM_IMAGES_DIR}/odroidc2/rootfs.cpio.gz")
    # Updated dtb based on initrd
    UpdateDtbFromInitrd(
        "${CAMKES_VM_IMAGES_DIR}/odroidc2/linux-dtb"
        ${rootfs_file}
        "0x2d700000"
        dtb_gen_target
        output_dtb_location
    )
endif()

AddCamkesCPPFlag(cpp_flags CONFIG_VARS VmEmmc2NoDMA VmVUSB Tk1DeviceFwd Tk1Insecure)

set(cpio_files "linux:${linux_image}" "linux-dtb:${output_dtb_location}")
if(rootfs_file)
    list(APPEND cpio_files "linux-initrd:${rootfs_file}")
endif()

DefineCAmkESVMFileServer(FILES ${cpio_files} DEPENDS dtb_gen_target)

CAmkESAddImportPath(${KernelARMPlatform})

# Declare root server
DeclareCAmkESRootserver(
    vm_minimal.camkes
    CPP_FLAGS
    ${cpp_flags}
    CPP_INCLUDES
    ${CAMKES_VM_DIR}/components/VM_Arm
)
