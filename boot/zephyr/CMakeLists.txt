# CMakeLists.txt for building mcuboot as a Zephyr project
#
# Copyright (c) 2017 Open Source Foundries Limited
# Copyright (c) 2023-2025 Nordic Semiconductor ASA
#
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.13.1)

# This sample shows usage of an external module and we need to set the
# set the extra module path before calling find_package(Zephyr).
if(TEST_RUNTIME_SOURCE_HOOKS)
  set(EXTRA_ZEPHYR_MODULES "${CMAKE_SOURCE_DIR}/../../samples/runtime-source/zephyr/hooks")
endif()

# find_package(Zephyr) in order to load application boilerplate:
# http://docs.zephyrproject.org/application/application.html
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(NONE)

# Path to "boot" subdirectory of repository root.
get_filename_component(BOOT_DIR ${APPLICATION_SOURCE_DIR} DIRECTORY)
# Path to top-level repository root directory.
get_filename_component(MCUBOOT_DIR ${BOOT_DIR} DIRECTORY)
# Path to tinycrypt library source subdirectory of MCUBOOT_DIR.
set(TINYCRYPT_DIR "${MCUBOOT_DIR}/ext/tinycrypt/lib")
assert_exists(TINYCRYPT_DIR)
set(TINYCRYPT_SHA512_DIR "${MCUBOOT_DIR}/ext/tinycrypt-sha512/lib")
assert_exists(TINYCRYPT_SHA512_DIR)
# Path to crypto-fiat
set(FIAT_DIR "${MCUBOOT_DIR}/ext/fiat")
assert_exists(FIAT_DIR)
# Path to mbed-tls' asn1 parser library.
if(NOT CONFIG_MBEDTLS_BUILTIN AND NOT CONFIG_BOOT_KEY_IMPORT_BYPASS_ASN)
  set(MBEDTLS_ASN1_DIR "${MCUBOOT_DIR}/ext/mbedtls-asn1")
  assert_exists(MBEDTLS_ASN1_DIR)
endif()
set(NRF_DIR "${MCUBOOT_DIR}/ext/nrf")

if(CONFIG_BOOT_USE_NRF_CC310_BL)
set(NRFXLIB_DIR ${ZEPHYR_BASE}/../nrfxlib)
if(NOT EXISTS ${NRFXLIB_DIR})
  message(FATAL_ERROR "
    ------------------------------------------------------------------------
    No such file or directory: ${NRFXLIB_DIR}
    The current configuration enables nRF CC310 crypto accelerator hardware
    with the `CONFIG_BOOT_USE_NRF_CC310_BL` option. Please follow
    `ext/nrf/README.md` guide to fix your setup or use tinycrypt instead of
    the HW accelerator.
    To use the tinycrypt set `CONFIG_BOOT_ECDSA_TINYCRYPT` to y.
    ------------------------------------------------------------------------")
endif()
# Don't include this if we are using west
 add_subdirectory(${NRFXLIB_DIR} ${PROJECT_BINARY_DIR}/nrfxlib)
endif()

zephyr_library_include_directories(
  include
  )

if(DEFINED CONFIG_MBEDTLS)
  zephyr_library_include_directories(
    ${ZEPHYR_MBEDTLS_MODULE_DIR}/include
  )
endif()

# Zephyr port-specific sources.
zephyr_library_sources(
  main.c
  io.c
  flash_map_extended.c
  os.c
  keys.c
  )

if(DEFINED CONFIG_ENABLE_MGMT_PERUSER)
  zephyr_library_sources(
    boot_serial_extensions.c
    )

  zephyr_linker_sources_ifdef(
    CONFIG_ENABLE_MGMT_PERUSER
    SECTIONS include/boot_serial/boot_serial.ld
  )

  if(DEFINED CONFIG_BOOT_MGMT_CUSTOM_STORAGE_ERASE)
    zephyr_library_sources(
      boot_serial_extension_zephyr_basic.c
      )
  endif()
endif()

if(DEFINED CONFIG_BOOT_SHARE_BACKEND_RETENTION)
  zephyr_library_sources(
    shared_data.c
    )
endif()

# Generic bootutil sources and includes.
zephyr_library_include_directories(${BOOT_DIR}/bootutil/include)
zephyr_library_sources(
  ${BOOT_DIR}/bootutil/src/image_validate.c
  ${BOOT_DIR}/bootutil/src/tlv.c
  ${BOOT_DIR}/bootutil/src/encrypted.c
  ${BOOT_DIR}/bootutil/src/image_rsa.c
  ${BOOT_DIR}/bootutil/src/image_ecdsa.c
  ${BOOT_DIR}/bootutil/src/image_ed25519.c
  ${BOOT_DIR}/bootutil/src/bootutil_misc.c
  ${BOOT_DIR}/bootutil/src/fault_injection_hardening.c
  )

if(DEFINED CONFIG_BOOT_ENCRYPT_X25519 AND DEFINED CONFIG_BOOT_ED25519_PSA)
  zephyr_library_sources(${BOOT_DIR}/bootutil/src/encrypted_psa.c)
endif()

if(DEFINED CONFIG_MEASURED_BOOT OR DEFINED CONFIG_BOOT_SHARE_DATA)
  zephyr_library_sources(
    ${BOOT_DIR}/bootutil/src/boot_record.c
    )

  # Set a define for this file which will allow inclusion of the Zephyr version
  # include file
  set_source_files_properties(
    ${BOOT_DIR}/bootutil/src/boot_record.c
    PROPERTIES COMPILE_FLAGS -DZEPHYR_VER_INCLUDE=1
    )
endif()

# library which might be common source code for MCUBoot and an application
zephyr_link_libraries(MCUBOOT_BOOTUTIL)

if(CONFIG_BOOT_FIH_PROFILE_HIGH)
zephyr_library_sources(
  ${BOOT_DIR}/bootutil/src/fault_injection_hardening_delay_rng_mbedtls.c
  )
endif()

if(CONFIG_SINGLE_APPLICATION_SLOT_RAM_LOAD)
  zephyr_library_sources(
    ${BOOT_DIR}/zephyr/single_loader.c
    ${BOOT_DIR}/bootutil/src/ram_load.c
    )
  zephyr_library_include_directories(${BOOT_DIR}/bootutil/src)
elseif(CONFIG_SINGLE_APPLICATION_SLOT)
  zephyr_library_sources(
    ${BOOT_DIR}/zephyr/single_loader.c
    )
  zephyr_library_include_directories(${BOOT_DIR}/bootutil/src)
elseif(CONFIG_BOOT_FIRMWARE_LOADER)
  zephyr_library_sources(
    ${BOOT_DIR}/zephyr/firmware_loader.c
    )
  zephyr_library_include_directories(${BOOT_DIR}/bootutil/src)
else()
  zephyr_library_sources(
    ${BOOT_DIR}/bootutil/src/loader.c
    ${BOOT_DIR}/bootutil/src/swap_misc.c
    ${BOOT_DIR}/bootutil/src/caps.c
    )

  if(CONFIG_BOOT_SWAP_USING_MOVE)
    zephyr_library_sources(
      ${BOOT_DIR}/bootutil/src/swap_move.c
      )
  elseif(CONFIG_BOOT_SWAP_USING_OFFSET)
    zephyr_library_sources(
      ${BOOT_DIR}/bootutil/src/swap_offset.c
      )
  else()
    zephyr_library_sources(
      ${BOOT_DIR}/bootutil/src/swap_scratch.c
      )

    if(CONFIG_BOOT_RAM_LOAD)
      zephyr_library_sources(
        ${BOOT_DIR}/bootutil/src/ram_load.c
        )
    endif()
  endif()
endif()

if(CONFIG_BOOT_SIGNATURE_TYPE_ECDSA_P256 OR CONFIG_BOOT_ENCRYPT_EC256)
  if(MBEDTLS_ASN1_DIR)
    zephyr_library_include_directories(
      ${MBEDTLS_ASN1_DIR}/include
    )
    zephyr_library_sources(
      # Additionally pull in just the ASN.1 parser from mbedTLS.
      ${MBEDTLS_ASN1_DIR}/src/asn1parse.c
      ${MBEDTLS_ASN1_DIR}/src/platform_util.c
    )
  endif()
  if(CONFIG_BOOT_USE_TINYCRYPT)
    # When using ECDSA signatures, pull in our copy of the tinycrypt library.
    zephyr_library_include_directories(
      ${BOOT_DIR}/zephyr/include
      ${TINYCRYPT_DIR}/include
    )
    zephyr_include_directories(${TINYCRYPT_DIR}/include)

    zephyr_library_sources(
      ${TINYCRYPT_DIR}/source/ecc.c
      ${TINYCRYPT_DIR}/source/ecc_dsa.c
      ${TINYCRYPT_DIR}/source/sha256.c
      ${TINYCRYPT_DIR}/source/utils.c
    )
  elseif(CONFIG_BOOT_USE_NRF_CC310_BL)
    zephyr_library_sources(${NRF_DIR}/cc310_glue.c)
    zephyr_library_include_directories(${NRF_DIR})
    zephyr_link_libraries(nrfxlib_crypto)
  endif()

  if(CONFIG_MBEDTLS_CFG_FILE)
    # Since here we are not using Zephyr's mbedTLS but rather our own, we need
    # to set MBEDTLS_CONFIG_FILE ourselves. When using Zephyr's copy, this
    # variable is set by its Kconfig in the Zephyr codebase.
    zephyr_library_compile_definitions(
      MBEDTLS_CONFIG_FILE="${CONFIG_MBEDTLS_CFG_FILE}"
    )
  endif()
elseif(CONFIG_BOOT_SIGNATURE_TYPE_NONE)
  zephyr_library_include_directories(
    ${BOOT_DIR}/zephyr/include
    ${TINYCRYPT_DIR}/include
    )

  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/sha256.c
    ${TINYCRYPT_DIR}/source/utils.c
    )
elseif(CONFIG_BOOT_SIGNATURE_TYPE_RSA)
  # Use mbedTLS provided by Zephyr for RSA signatures. (Its config file
  # is set using Kconfig.)
  zephyr_include_directories(include)
  if(CONFIG_BOOT_ENCRYPT_RSA)
    set_source_files_properties(
      ${BOOT_DIR}/bootutil/src/encrypted.c
      PROPERTIES
      INCLUDE_DIRECTORIES ${ZEPHYR_MBEDTLS_MODULE_DIR}/library
      )
  endif()
elseif(CONFIG_BOOT_SIGNATURE_TYPE_ED25519 OR CONFIG_BOOT_ENCRYPT_X25519)
  if(CONFIG_BOOT_USE_TINYCRYPT)
    if(MBEDTLS_ASN1_DIR)
      zephyr_library_include_directories(
        ${MBEDTLS_ASN1_DIR}/include
      )
      zephyr_library_sources(
        # Additionally pull in just the ASN.1 parser from mbedTLS.
        ${MBEDTLS_ASN1_DIR}/src/asn1parse.c
        ${MBEDTLS_ASN1_DIR}/src/platform_util.c
      )
    endif()
    zephyr_library_include_directories(
      ${BOOT_DIR}/zephyr/include
      ${TINYCRYPT_DIR}/include
      ${TINYCRYPT_SHA512_DIR}/include
      )
    zephyr_library_sources(
      ${TINYCRYPT_DIR}/source/sha256.c
      ${TINYCRYPT_DIR}/source/utils.c
      ${TINYCRYPT_SHA512_DIR}/source/sha512.c
      )
    zephyr_library_compile_definitions(
      MBEDTLS_CONFIG_FILE="${CONFIG_MBEDTLS_CFG_FILE}"
      )
  else()
    zephyr_include_directories(include)
  endif()

  zephyr_library_include_directories(
    ${BOOT_DIR}/zephyr/include
    ${FIAT_DIR}/include/
  )

  if(NOT CONFIG_BOOT_ED25519_PSA)
    zephyr_library_sources(
      ${FIAT_DIR}/src/curve25519.c
    )
  else()
    zephyr_library_sources(
      ${MBEDTLS_ASN1_DIR}/src/asn1parse.c
      ${BOOT_DIR}/bootutil/src/ed25519_psa.c
    )
  endif()
endif()

if(NOT CONFIG_BOOT_ED25519_PSA)
  if(CONFIG_BOOT_ENCRYPT_EC256 OR CONFIG_BOOT_ENCRYPT_X25519)
    zephyr_library_sources(
      ${TINYCRYPT_DIR}/source/aes_encrypt.c
      ${TINYCRYPT_DIR}/source/aes_decrypt.c
      ${TINYCRYPT_DIR}/source/ctr_mode.c
      ${TINYCRYPT_DIR}/source/hmac.c
      ${TINYCRYPT_DIR}/source/ecc_dh.c
    )
  endif()
endif()

if(CONFIG_BOOT_ENCRYPT_EC256)
  zephyr_library_sources(
    ${TINYCRYPT_DIR}/source/ecc_dh.c
    )
endif()

if(CONFIG_MCUBOOT_SERIAL)
  zephyr_sources(${BOOT_DIR}/zephyr/serial_adapter.c)
  zephyr_sources(${BOOT_DIR}/boot_serial/src/boot_serial.c)
  zephyr_sources(${BOOT_DIR}/boot_serial/src/zcbor_bulk.c)

  zephyr_include_directories(${BOOT_DIR}/bootutil/include)
  zephyr_include_directories(${BOOT_DIR}/boot_serial/include)
  zephyr_include_directories(include)

  zephyr_include_directories_ifdef(
    CONFIG_BOOT_ERASE_PROGRESSIVELY
    ${BOOT_DIR}/bootutil/src
    )

  if(CONFIG_BOOT_ENCRYPT_IMAGE)
    zephyr_library_sources(
      ${BOOT_DIR}/boot_serial/src/boot_serial_encryption.c
    )
  endif()
endif()

if(NOT CONFIG_BOOT_SIGNATURE_KEY_FILE STREQUAL "")
  # CONF_FILE points to the KConfig configuration files of the bootloader.
  foreach (filepath ${CONF_FILE})
    file(READ ${filepath} temp_text)
    string(FIND "${temp_text}" ${CONFIG_BOOT_SIGNATURE_KEY_FILE} match)
    if (${match} GREATER_EQUAL 0)
      if (NOT DEFINED CONF_DIR)
        get_filename_component(CONF_DIR ${filepath} DIRECTORY)
      else()
        message(FATAL_ERROR "Signature key file defined in multiple conf files")
      endif()
    endif()
  endforeach()

  if(IS_ABSOLUTE ${CONFIG_BOOT_SIGNATURE_KEY_FILE})
    set(KEY_FILE ${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  elseif((DEFINED CONF_DIR) AND
	 (EXISTS ${CONF_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE}))
    set(KEY_FILE ${CONF_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  else()
    set(KEY_FILE ${MCUBOOT_DIR}/${CONFIG_BOOT_SIGNATURE_KEY_FILE})
  endif()
  message("MCUBoot bootloader key file: ${KEY_FILE}")

  set(mcuboot_default_signature_files
      ${MCUBOOT_DIR}/root-ec-p256-pkcs8.pem
      ${MCUBOOT_DIR}/root-ec-p384.pem
      ${MCUBOOT_DIR}/root-ec-p384-pkcs8.pem
      ${MCUBOOT_DIR}/root-ed25519.pem
      ${MCUBOOT_DIR}/root-rsa-2048.pem
      ${MCUBOOT_DIR}/root-rsa-3072.pem
      ${MCUBOOT_DIR}/root-ec-p256.pem
  )

  # Emit a warning if using one of the default MCUboot key files
  if(${KEY_FILE} IN_LIST mcuboot_default_signature_files)
    message(WARNING "WARNING: Using default MCUboot signing key file, this file is for debug use only and is not secure!")
  endif()

  set(GENERATED_PUBKEY ${ZEPHYR_BINARY_DIR}/autogen-pubkey.c)
  add_custom_command(
    OUTPUT ${GENERATED_PUBKEY}
    COMMAND
    ${PYTHON_EXECUTABLE}
    ${MCUBOOT_DIR}/scripts/imgtool.py
    getpub
    -k
    ${KEY_FILE}
    > ${GENERATED_PUBKEY}
    DEPENDS ${KEY_FILE}
    )
  zephyr_library_sources(${GENERATED_PUBKEY})
endif()

if(CONFIG_BOOT_ENCRYPTION_KEY_FILE AND NOT CONFIG_BOOT_ENCRYPTION_KEY_FILE STREQUAL "")
  # CONF_FILE points to the KConfig configuration files of the bootloader.
  unset(CONF_DIR)
  foreach(filepath ${CONF_FILE})
    file(READ ${filepath} temp_text)
    string(FIND "${temp_text}" ${CONFIG_BOOT_ENCRYPTION_KEY_FILE} match)
    if(${match} GREATER_EQUAL 0)
      if(NOT DEFINED CONF_DIR)
        get_filename_component(CONF_DIR ${filepath} DIRECTORY)
      else()
        message(FATAL_ERROR "Encryption key file defined in multiple conf files")
      endif()
    endif()
  endforeach()

  if(IS_ABSOLUTE ${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
    set(KEY_FILE ${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  elseif((DEFINED CONF_DIR) AND
	 (EXISTS ${CONF_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE}))
    set(KEY_FILE ${CONF_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  else()
    set(KEY_FILE ${MCUBOOT_DIR}/${CONFIG_BOOT_ENCRYPTION_KEY_FILE})
  endif()
  message("MCUBoot bootloader encryption key file: ${KEY_FILE}")

  # Emit a warning if using one of the default MCUboot key files
  set(mcuboot_default_encryption_files
      ${MCUBOOT_DIR}/enc-ec256-priv.pem
      ${MCUBOOT_DIR}/enc-ec256-pub.pem
      ${MCUBOOT_DIR}/enc-rsa2048-priv.pem
      ${MCUBOOT_DIR}/enc-rsa2048-pub.pem
      ${MCUBOOT_DIR}/enc-x25519-priv.pem
      ${MCUBOOT_DIR}/enc-x25519-pub.pem
  )

  if(${KEY_FILE} IN_LIST mcuboot_default_encryption_files)
    message(WARNING "WARNING: Using default MCUboot encryption key file, this file is for debug use only and is not secure!")
  endif()

  set(GENERATED_ENCKEY ${ZEPHYR_BINARY_DIR}/autogen-enckey.c)
  add_custom_command(
    OUTPUT ${GENERATED_ENCKEY}
    COMMAND
    ${PYTHON_EXECUTABLE}
    ${MCUBOOT_DIR}/scripts/imgtool.py
    getpriv
    -k
    ${KEY_FILE}
    > ${GENERATED_ENCKEY}
    DEPENDS ${KEY_FILE}
    )
  zephyr_library_sources(${GENERATED_ENCKEY})
endif()

if(CONFIG_MCUBOOT_CLEANUP_ARM_CORE)
  # ARM Cortex-M
  zephyr_library_sources_ifdef(
    CONFIG_CPU_CORTEX_M
    ${BOOT_DIR}/zephyr/cleanup/arm_cortex_m.c
  )

  # ARM Cortex-R
  zephyr_library_sources_ifdef(
    CONFIG_ARMV7_R
    ${BOOT_DIR}/zephyr/cleanup/arm_cortex_r.c
  )
endif()

if(CONFIG_MCUBOOT_BOOT_BANNER)
  # Replace Zephyr's boot banner with the MCUboot one
  zephyr_sources(kernel/banner.c)
endif()

function(align_up num align result)
  math(EXPR out "(((${num}) + ((${align}) - 1)) & ~((${align}) - 1))")
  set(${result} "${out}" PARENT_SCOPE)
endfunction()

function(dt_get_parent node)
  string(FIND "${${node}}" "/" pos REVERSE)

  if(pos EQUAL -1)
    message(FATAL_ERROR "Unable to get parent of node: ${${node}}")
  endif()

  string(SUBSTRING "${${node}}" 0 ${pos} ${node})
  set(${node} "${${node}}" PARENT_SCOPE)
endfunction()

dt_nodelabel(slot0_flash NODELABEL "slot0_partition")
dt_prop(slot0_size PATH "${slot0_flash}" PROPERTY "reg" INDEX 1)
dt_get_parent(slot0_flash)
dt_get_parent(slot0_flash)
dt_prop(erase_size_slot0 PATH "${slot0_flash}" PROPERTY "erase-block-size")
dt_prop(write_size_slot0 PATH "${slot0_flash}" PROPERTY "write-block-size")

if(CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET)
  if(DEFINED erase_size_slot0)
    zephyr_compile_definitions("MCUBOOT_SLOT0_EXPECTED_ERASE_SIZE=${erase_size_slot0}")
  endif()

  if(DEFINED write_size_slot0)
    zephyr_compile_definitions("MCUBOOT_SLOT0_EXPECTED_WRITE_SIZE=${write_size_slot0}")
  endif()
endif()

if(NOT CONFIG_SINGLE_APPLICATION_SLOT AND NOT CONFIG_SINGLE_APPLICATION_SLOT_RAM_LOAD)
  dt_nodelabel(slot1_flash NODELABEL "slot1_partition")
  dt_prop(slot1_size PATH "${slot1_flash}" PROPERTY "reg" INDEX 1)
  dt_get_parent(slot1_flash)
  dt_get_parent(slot1_flash)
  dt_prop(erase_size_slot1 PATH "${slot1_flash}" PROPERTY "erase-block-size")
  dt_prop(write_size_slot1 PATH "${slot1_flash}" PROPERTY "write-block-size")

  if(CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET)
    if(DEFINED erase_size_slot1)
      zephyr_compile_definitions("MCUBOOT_SLOT1_EXPECTED_ERASE_SIZE=${erase_size_slot1}")
    endif()

    if(DEFINED write_size_slot1)
      zephyr_compile_definitions("MCUBOOT_SLOT1_EXPECTED_WRITE_SIZE=${write_size_slot1}")
    endif()
  endif()
endif()

if(CONFIG_BOOT_MAX_IMG_SECTORS_AUTO)
  if(NOT DEFINED slot0_size)
    message(WARNING "Unable to determine size of slot0 partition, cannot calculate minimum sector usage")
  elseif(NOT DEFINED erase_size_slot0)
    message(WARNING "Unable to determine erase size of slot0 partition, cannot calculate minimum sector usage")
  else()
    math(EXPR slot_min_sectors "${slot0_size} / ${erase_size_slot0}")
  endif()

  if(NOT CONFIG_SINGLE_APPLICATION_SLOT AND NOT CONFIG_SINGLE_APPLICATION_SLOT_RAM_LOAD)
    if(NOT DEFINED slot1_size)
      message(WARNING "Unable to determine size of slot1 partition, cannot calculate minimum sector usage")
    elseif(NOT DEFINED erase_size_slot1)
      message(WARNING "Unable to determine erase size of slot1 partition, cannot calculate minimum sector usage")
    else()
      math(EXPR slot1_min_sectors "${slot1_size} / ${erase_size_slot1}")

      if("${slot1_min_sectors}" GREATER "${slot_min_sectors}")
        set(slot_min_sectors ${slot1_min_sectors})
      endif()
    endif()
  endif()

  if(DEFINED slot_min_sectors AND "${slot_min_sectors}" GREATER "0")
    zephyr_compile_definitions("MIN_SECTOR_COUNT=${slot_min_sectors}")
    message("Calculated maximum number of sectors: ${slot_min_sectors}")
  else()
    message(WARNING "Unable to calculate minimum number of sector sizes, falling back to 128 sector default. Please disable CONFIG_BOOT_MAX_IMG_SECTORS_AUTO and set CONFIG_BOOT_MAX_IMG_SECTORS to the required value")
  endif()
endif()

if((CONFIG_BOOT_SWAP_USING_SCRATCH OR CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET) AND (DEFINED write_size_slot0 OR DEFINED write_size_slot1))
  zephyr_library_sources(flash_check.c)
endif()

if(CONFIG_BOOT_RAM_LOAD)
  zephyr_library_sources(ram_load.c)
endif()

if(SYSBUILD)
  if(CONFIG_SINGLE_APPLICATION_SLOT OR CONFIG_BOOT_FIRMWARE_LOADER OR CONFIG_BOOT_SWAP_USING_SCRATCH OR CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET OR CONFIG_BOOT_UPGRADE_ONLY OR CONFIG_BOOT_DIRECT_XIP OR CONFIG_BOOT_RAM_LOAD)
    # TODO: RAM LOAD support
    dt_nodelabel(slot0_flash NODELABEL "slot0_partition")
    dt_get_parent(slot0_flash)
    dt_get_parent(slot0_flash)

    if(NOT CONFIG_SINGLE_APPLICATION_SLOT)
      dt_nodelabel(slot1_flash NODELABEL "slot1_partition")
      dt_get_parent(slot1_flash)
      dt_get_parent(slot1_flash)

      if(NOT "${slot0_flash}" STREQUAL "${slot1_flash}")
        # Check both slots for the one with the largest write/erase block size
        dt_prop(erase_size_slot0 PATH "${slot0_flash}" PROPERTY "erase-block-size")
        dt_prop(write_size_slot0 PATH "${slot0_flash}" PROPERTY "write-block-size")
        dt_prop(erase_size_slot1 PATH "${slot1_flash}" PROPERTY "erase-block-size")
        dt_prop(write_size_slot1 PATH "${slot1_flash}" PROPERTY "write-block-size")

        if(DEFINED erase_size_slot0 AND DEFINED erase_size_slot1)
          if(${erase_size_slot0} GREATER ${erase_size_slot1})
            set(erase_size ${erase_size_slot0})
          else()
            set(erase_size ${erase_size_slot1})
          endif()
        elseif(DEFINED erase_size_slot0)
          set(erase_size ${erase_size_slot0})
        elseif(DEFINED erase_size_slot1)
          set(erase_size ${erase_size_slot1})
        endif()

        if(DEFINED write_size_slot0 AND DEFINED write_size_slot1)
          if(${write_size_slot0} GREATER ${write_size_slot1})
            set(write_size ${write_size_slot0})
          else()
            set(write_size ${write_size_slot1})
          endif()
        elseif(DEFINED write_size_slot0)
          set(write_size ${write_size_slot0})
        elseif(DEFINED write_size_slot1)
          set(write_size ${write_size_slot1})
        endif()
      else()
        dt_prop(erase_size PATH "${slot0_flash}" PROPERTY "erase-block-size")
        dt_prop(write_size PATH "${slot0_flash}" PROPERTY "write-block-size")
      endif()
    else()
      dt_prop(erase_size PATH "${slot0_flash}" PROPERTY "erase-block-size")
      dt_prop(write_size PATH "${slot0_flash}" PROPERTY "write-block-size")
    endif()

    if(NOT DEFINED erase_size)
      message(WARNING "Unable to determine erase size of slot0 or slot1 partition, setting to 1 (this is probably wrong)")
      set(erase_size 1)
    endif()

    if(NOT DEFINED write_size)
      message(WARNING "Unable to determine write size of slot0 or slot1 partition, setting to 8 (this is probably wrong)")
      set(write_size 8)
    endif()

    if(${write_size} LESS 8)
      set(max_align_size 8)
    else()
      set(max_align_size ${write_size})
    endif()

    set(key_size 0)

    # Boot trailer magic size
    set(boot_magic_size 16)

    # Estimates for trailer TLV data size, this was taken from hello world builds for nrf52840dk
    if(CONFIG_BOOT_SIGNATURE_TYPE_RSA)
      if(CONFIG_BOOT_SIGNATURE_TYPE_RSA_LEN EQUAL 3072)
        set(boot_tlv_estimate 464)
      else()
        set(boot_tlv_estimate 336)
      endif()
    elseif(CONFIG_BOOT_SIGNATURE_TYPE_ECDSA_P256)
      set(boot_tlv_estimate 150)
    elseif(CONFIG_BOOT_SIGNATURE_TYPE_ED25519)
      set(boot_tlv_estimate 144)
    else()
      set(boot_tlv_estimate 40)
    endif()

    if(CONFIG_BOOT_ENCRYPT_RSA OR CONFIG_BOOT_ENCRYPT_EC256 OR CONFIG_BOOT_ENCRYPT_X25519)
      # 128-bit AES key size
      set(boot_enc_key_size 16)

      if(CONFIG_BOOT_SWAP_SAVE_ENCTLV)
        if(CONFIG_BOOT_ENCRYPT_RSA)
          set(key_size 256)
        elseif(CONFIG_BOOT_ENCRYPT_EC256)
          math(EXPR key_size "65 + 32 + ${boot_enc_key_size}")
        elseif(CONFIG_BOOT_ENCRYPT_X25519)
          math(EXPR key_size "32 + 32 + ${boot_enc_key_size}")
        endif()
      else()
        set(key_size "${boot_enc_key_size}")
      endif()

      align_up(${key_size} ${max_align_size} key_size)
      math(EXPR key_size "${key_size} * 2")
    endif()

    align_up(${boot_magic_size} ${write_size} boot_magic_size)

    if(CONFIG_SINGLE_APPLICATION_SLOT OR CONFIG_BOOT_FIRMWARE_LOADER)
      set(boot_swap_data_size 0)
    else()
      math(EXPR boot_swap_data_size "${max_align_size} * 4")
    endif()

    if(CONFIG_BOOT_SWAP_USING_SCRATCH OR CONFIG_BOOT_SWAP_USING_MOVE)
      if(CONFIG_BOOT_MAX_IMG_SECTORS_AUTO AND DEFINED slot_min_sectors AND "${slot_min_sectors}" GREATER "0")
        math(EXPR boot_status_data_size "${slot_min_sectors} * (3 * ${write_size})")
      else()
        if(CONFIG_BOOT_MAX_IMG_SECTORS)
          math(EXPR boot_status_data_size "${CONFIG_BOOT_MAX_IMG_SECTORS} * (3 * ${write_size})")
        else()
          message(WARNING "CONFIG_BOOT_MAX_IMG_SECTORS is not defined, falling back to 128 sector default. Please set CONFIG_BOOT_MAX_IMG_SECTORS to the required value")
          math(EXPR boot_status_data_size "128 * (3 * ${write_size})")
        endif()
      endif()
    elseif(CONFIG_BOOT_SWAP_USING_OFFSET)
      if(CONFIG_BOOT_MAX_IMG_SECTORS_AUTO AND DEFINED slot_min_sectors AND "${slot_min_sectors}" GREATER "0")
        math(EXPR boot_status_data_size "${slot_min_sectors} * (2 * ${write_size})")
      else()
        if(CONFIG_BOOT_MAX_IMG_SECTORS)
          math(EXPR boot_status_data_size "${CONFIG_BOOT_MAX_IMG_SECTORS} * (2 * ${write_size})")
        else()
          message(WARNING "CONFIG_BOOT_MAX_IMG_SECTORS is not defined, falling back to 128 sector default. Please set CONFIG_BOOT_MAX_IMG_SECTORS to the required value")
          math(EXPR boot_status_data_size "128 * (2 * ${write_size})")
        endif()
      endif()
    else()
      set(boot_status_data_size 0)
    endif()

    math(EXPR trailer_size "${key_size} + ${boot_magic_size} + ${boot_swap_data_size} + ${boot_status_data_size}")

    if(CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET)
      align_up(${trailer_size} ${erase_size} trailer_size)
    endif()

    math(EXPR required_size "${trailer_size} + ${boot_tlv_estimate}")

    if(CONFIG_SINGLE_APPLICATION_SLOT OR CONFIG_BOOT_FIRMWARE_LOADER)
      set(required_upgrade_size "0")
    else()
      math(EXPR required_upgrade_size "${boot_magic_size} + ${boot_swap_data_size} + ${boot_status_data_size}")

      if(CONFIG_BOOT_SWAP_USING_MOVE OR CONFIG_BOOT_SWAP_USING_OFFSET)
        align_up(${required_upgrade_size} ${erase_size} required_upgrade_size)
      endif()
    endif()

    if(CONFIG_BOOT_SWAP_USING_MOVE)
      math(EXPR required_size "${required_size} + ${erase_size}")
      math(EXPR required_upgrade_size "${required_upgrade_size} + ${erase_size}")
    elseif(CONFIG_BOOT_SWAP_USING_OFFSET)
#todo: check how different slot sizes are...
#      math(EXPR required_size "${required_size} + ${erase_size}")
#      math(EXPR required_upgrade_size "${required_upgrade_size} + ${erase_size}")
    endif()
  else()
    set(required_size 0)
    set(required_upgrade_size 0)
  endif()

  set(mcuboot_image_footer_size ${required_size} CACHE INTERNAL "Estimated MCUboot image trailer size" FORCE)
  set(mcuboot_image_upgrade_footer_size ${required_upgrade_size} CACHE INTERNAL "Estimated MCUboot update image trailer size" FORCE)
endif()
