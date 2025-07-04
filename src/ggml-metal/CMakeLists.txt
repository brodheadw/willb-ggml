find_library(FOUNDATION_LIBRARY Foundation REQUIRED)
find_library(METAL_FRAMEWORK    Metal      REQUIRED)
find_library(METALKIT_FRAMEWORK MetalKit   REQUIRED)

message(STATUS "Metal framework found")

# Declare the Metal backend library
ggml_add_backend_library(ggml-metal
    ggml-metal.m
)

target_link_libraries(ggml-metal PRIVATE
    ${FOUNDATION_LIBRARY}
    ${METAL_FRAMEWORK}
    ${METALKIT_FRAMEWORK}
)

# Include this directory for access to ggml-metal-impl.h
target_include_directories(ggml-metal
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}
)

if (GGML_METAL_NDEBUG)
    add_compile_definitions(GGML_METAL_NDEBUG)
endif()

if (GGML_METAL_USE_BF16)
    add_compile_definitions(GGML_METAL_USE_BF16)
endif()

# Copy Metal source files to the binary output directory
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/../ggml-common.h
    ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-common.h
    COPYONLY
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/ggml-metal.metal
    ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal
    COPYONLY
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/ggml-metal-impl.h
    ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal-impl.h
    COPYONLY
)

# Setup Metal shader compilation
set(METALLIB_COMMON "${CMAKE_CURRENT_SOURCE_DIR}/../ggml-common.h")

if (GGML_METAL_EMBED_LIBRARY)
    enable_language(ASM)
    add_compile_definitions(GGML_METAL_EMBED_LIBRARY)

    set(METALLIB_SOURCE "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal")
    set(METALLIB_IMPL   "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal-impl.h")

    file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/autogenerated")

    set(METALLIB_EMBED_ASM        "${CMAKE_BINARY_DIR}/autogenerated/ggml-metal-embed.s")
    set(METALLIB_SOURCE_EMBED     "${CMAKE_BINARY_DIR}/autogenerated/ggml-metal-embed.metal")
    set(METALLIB_SOURCE_EMBED_TMP "${CMAKE_BINARY_DIR}/autogenerated/ggml-metal-embed.metal.tmp")

    add_custom_command(
        OUTPUT "${METALLIB_EMBED_ASM}"
        COMMAND echo "Embedding Metal library"
        COMMAND sed -e "/__embed_ggml-common.h__/r ${METALLIB_COMMON}"       -e "/__embed_ggml-common.h__/d"         < "${METALLIB_SOURCE}"           > "${METALLIB_SOURCE_EMBED_TMP}"
        COMMAND sed -e "/\\#include \\\"ggml-metal-impl.h\\\"/r ${METALLIB_IMPL}" -e "/\\#include \\\"ggml-metal-impl.h\\\"/d" < "${METALLIB_SOURCE_EMBED_TMP}" > "${METALLIB_SOURCE_EMBED}"
        COMMAND echo ".section __DATA,__ggml_metallib"          >  "${METALLIB_EMBED_ASM}"
        COMMAND echo ".globl _ggml_metallib_start"              >> "${METALLIB_EMBED_ASM}"
        COMMAND echo "_ggml_metallib_start:"                    >> "${METALLIB_EMBED_ASM}"
        COMMAND echo .incbin "\\\"${METALLIB_SOURCE_EMBED}\\\"" >> "${METALLIB_EMBED_ASM}"
        COMMAND echo ".globl _ggml_metallib_end"                >> "${METALLIB_EMBED_ASM}"
        COMMAND echo "_ggml_metallib_end:"                      >> "${METALLIB_EMBED_ASM}"
        DEPENDS ${METALLIB_COMMON} ${METALLIB_SOURCE} ${METALLIB_IMPL}
        COMMENT "Generate assembly for embedded Metal library"
        VERBATIM
    )

    target_sources(ggml-metal PRIVATE "${METALLIB_EMBED_ASM}")

else()
    if (GGML_METAL_SHADER_DEBUG)
        set(XC_FLAGS -fno-fast-math -fno-inline -g)
    else()
        set(XC_FLAGS -O3)
    endif()

    if (GGML_METAL_MACOSX_VERSION_MIN)
        message(STATUS "Adding  -mmacosx-version-min=${GGML_METAL_MACOSX_VERSION_MIN} flag to metal compilation")
        list(APPEND XC_FLAGS -mmacosx-version-min=${GGML_METAL_MACOSX_VERSION_MIN})
    endif()

    if (GGML_METAL_STD)
        message(STATUS "Adding  -std=${GGML_METAL_STD} flag to metal compilation")
        list(APPEND XC_FLAGS -std=${GGML_METAL_STD})
    endif()

    add_custom_command(
        OUTPUT ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
        COMMAND xcrun -sdk macosx metal ${XC_FLAGS} -c ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal -o - |
            xcrun -sdk macosx metallib - -o ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
        DEPENDS ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ggml-metal.metal ${METALLIB_COMMON}
        COMMENT "Compiling Metal kernels"
    )

    add_custom_target(
        ggml-metal-lib ALL
        DEPENDS ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
    )
endif()

if (NOT GGML_METAL_EMBED_LIBRARY)
    install(
        FILES ${CMAKE_CURRENT_SOURCE_DIR}/ggml-metal.metal
        PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ
        DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

    install(
        FILES ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/default.metallib
        DESTINATION ${CMAKE_INSTALL_BINDIR}
    )
endif()