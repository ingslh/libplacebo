# src/CMakeLists.txt

# --- Dependencies Setup ---
set(PL_DEPS Threads::Threads)
if(UNIX)
    list(APPEND PL_DEPS m)
endif()

# Include directories
set(PL_INC_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR} # for config.h
    ${CMAKE_CURRENT_BINARY_DIR}/include # for config.h if installed there
)

# 3rdparty: fast_float
if(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/fast_float/include")
    list(APPEND PL_INC_DIRS "${CMAKE_SOURCE_DIR}/3rdparty/fast_float/include")
endif()

# 3rdparty: glad
if(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/glad/include")
    list(APPEND PL_INC_DIRS "${CMAKE_SOURCE_DIR}/3rdparty/glad/include")
    # Glad sources are usually generated or included? 
    # meson says: dependencies: build_deps + glad_dep. 
    # glad_dep is not defined in the snippets I read.
    # Usually libplacebo embeds glad?
    # I'll check if src/opengl/include/glad/meson.build defines it.
    # For now assume header only or handled in opengl.
endif()

# --- Preprocessing Setup ---
set(PYTHON_ENV_PATH 
    "${CMAKE_SOURCE_DIR}/3rdparty/jinja/src"
    "${CMAKE_SOURCE_DIR}/3rdparty/markupsafe/src"
    "${CMAKE_SOURCE_DIR}/3rdparty/glad"
)
if(WIN32)
    string(JOIN ";" PYTHON_PATH_STR ${PYTHON_ENV_PATH})
else()
    string(JOIN ":" PYTHON_PATH_STR ${PYTHON_ENV_PATH})
endif()

macro(pl_glsl_preproc filename subdir tgt)
    set(_input "${CMAKE_CURRENT_SOURCE_DIR}/${subdir}/${filename}")
    if("${subdir}" STREQUAL ".")
        set(_output "${CMAKE_CURRENT_BINARY_DIR}/${filename}")
    else()
        set(_output "${CMAKE_CURRENT_BINARY_DIR}/${subdir}/${filename}")
    endif()
    
    # Create directory if needed
    get_filename_component(_outdir ${_output} DIRECTORY)
    file(MAKE_DIRECTORY ${_outdir})

    set(_strip_arg "")
    if(NOT CMAKE_BUILD_TYPE MATCHES "Debug")
        set(_strip_arg "--strip")
    endif()

    add_custom_command(
        OUTPUT ${_output}
        COMMAND ${CMAKE_COMMAND} -E env "PYTHONPATH=${PYTHON_PATH_STR}"
                ${Python3_EXECUTABLE} "${CMAKE_SOURCE_DIR}/tools/glsl_preproc/main.py"
                ${_strip_arg}
                ${_input} ${_output}
        DEPENDS ${_input} "${CMAKE_SOURCE_DIR}/tools/glsl_preproc/main.py"
        COMMENT "Preprocessing ${subdir}/${filename}"
    )
    # list(APPEND PL_SOURCES ${_output}) # Broken in subdirs
    target_sources(${tgt} PRIVATE ${_output})
endmacro()

# --- Sources ---
set(PL_SOURCES
    cache.c
    colorspace.c
    common.c
    convert.cc
    dither.c
    dispatch.c
    dummy.c
    filters.c
    format.c
    gamut_mapping.c
    gpu.c
    gpu/utils.c
    log.c
    options.c
    pl_alloc.c
    pl_string.c
    swapchain.c
    tone_mapping.c
    utils/dolbyvision.c
    utils/frame_queue.c
    utils/upload.c
    # Preprocessed sources will be added via macro
)

# Shaders
# Sources handled in src/shaders/CMakeLists.txt


# --- Feature Detection ---
set(PL_EXTRA_DEFS "")

# Helper macro for feature defines
macro(pl_add_feature name enabled)
    if(${enabled})
        string(APPEND PL_EXTRA_DEFS "#define PL_HAVE_${name} 1\n")
        set(PL_HAVE_${name} 1)
    else()
        string(APPEND PL_EXTRA_DEFS "#undef PL_HAVE_${name}\n")
        set(PL_HAVE_${name} 0)
    endif()
endmacro()

# LCMS
if(PL_USE_LCMS)
    find_path(LCMS2_INCLUDE_DIR lcms2.h)
    find_library(LCMS2_LIBRARY lcms2)
    if(LCMS2_INCLUDE_DIR AND LCMS2_LIBRARY)
        list(APPEND PL_DEPS ${LCMS2_LIBRARY})
        list(APPEND PL_INC_DIRS ${LCMS2_INCLUDE_DIR})
        pl_add_feature("LCMS" ON)
    else()
        pl_add_feature("LCMS" OFF)
    endif()
else()
    pl_add_feature("LCMS" OFF)
endif()

# XXHASH
if(PL_USE_XXHASH)
    find_path(XXHASH_INCLUDE_DIR xxhash.h)
    find_library(XXHASH_LIBRARY xxhash)
    if(XXHASH_INCLUDE_DIR AND XXHASH_LIBRARY)
         list(APPEND PL_DEPS ${XXHASH_LIBRARY})
         list(APPEND PL_INC_DIRS ${XXHASH_INCLUDE_DIR})
         pl_add_feature("XXHASH" ON)
    else()
         pl_add_feature("XXHASH" OFF)
    endif()
else()
    pl_add_feature("XXHASH" OFF)
endif()

# UNWIND (Linux mostly)
if(PL_USE_UNWIND)
    find_path(LIBUNWIND_INCLUDE_DIR libunwind.h)
    find_library(LIBUNWIND_LIBRARY unwind)
    if(LIBUNWIND_INCLUDE_DIR AND LIBUNWIND_LIBRARY)
        list(APPEND PL_DEPS ${LIBUNWIND_LIBRARY})
        list(APPEND PL_INC_DIRS ${LIBUNWIND_INCLUDE_DIR})
        set(PL_HAVE_UNWIND 1) # Internal config
    else()
        set(PL_HAVE_UNWIND 0)
    endif()
endif()

# D3D11
if(PL_USE_D3D11 AND WIN32)
    # Check headers... assuming available on Windows
    pl_add_feature("D3D11" ON)
    list(APPEND PL_SOURCES d3d11/swapchain.c d3d11/gpu.c d3d11/utils.c)
    # Dependencies: d3d11, dxgi, etc. usually implicit or find_library
else()
    pl_add_feature("D3D11" OFF)
endif()

# Vulkan
if(PL_USE_VULKAN)
    # Vulkan Headers (3rdparty or system)
    if(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/include")
        list(APPEND PL_INC_DIRS "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/include")
    endif()
    
    # Basic check - maybe find_package(Vulkan)
    find_package(Vulkan)
    if(Vulkan_FOUND OR EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/include")
        pl_add_feature("VULKAN" ON)
        list(APPEND PL_SOURCES 
            vulkan/command.c
            vulkan/context.c
            vulkan/formats.c
            vulkan/gpu.c
            vulkan/gpu_buf.c
            vulkan/gpu_pass.c
            vulkan/gpu_tex.c
            vulkan/malloc.c
            vulkan/swapchain.c
            vulkan/utils.c
            # vulkan/utils_gen.c # Generated
        )

        # Generate utils_gen.c
        set(VK_XML_PATH "")
        if(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/registry/vk.xml")
            set(VK_XML_PATH "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/registry/vk.xml")
        elseif(Vulkan_REGISTRY_XML)
             set(VK_XML_PATH "${Vulkan_REGISTRY_XML}")
        endif()
        
        if(VK_XML_PATH)
             # datadir arg: pass install prefix datadir? Or just a dummy if not used for much?
             # meson uses: get_option('prefix') / get_option('datadir')
             set(_datadir "${CMAKE_INSTALL_PREFIX}/share")
             
             add_custom_command(
                OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/vulkan/utils_gen.c
                COMMAND ${CMAKE_COMMAND} -E env "PYTHONPATH=${PYTHON_PATH_STR}"
                        ${Python3_EXECUTABLE} "${CMAKE_CURRENT_SOURCE_DIR}/vulkan/utils_gen.py"
                        "${_datadir}" "${VK_XML_PATH}"
                        "${CMAKE_CURRENT_BINARY_DIR}/vulkan/utils_gen.c"
                DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/vulkan/utils_gen.py" "${VK_XML_PATH}"
                COMMENT "Generating vulkan/utils_gen.c"
             )
             list(APPEND PL_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/vulkan/utils_gen.c)
        else()
             message(WARNING "vk.xml not found, Vulkan support might be incomplete")
        endif()

        # Link Vulkan if proc addr not disabled? 
        # Meson logic: vulkan_loader = dependency('vulkan', required: false)
        if(Vulkan_FOUND)
            list(APPEND PL_DEPS Vulkan::Vulkan)
        endif()
    else()
        pl_add_feature("VULKAN" OFF)
    endif()
else()
    pl_add_feature("VULKAN" OFF)
endif()

# OpenGL
if(PL_USE_OPENGL)
    pl_add_feature("OPENGL" ON)
    # opengl sources handled in src/opengl/CMakeLists.txt
    
    # Need to link dl on Linux if checking symbols?
    if(UNIX)
        list(APPEND PL_DEPS ${CMAKE_DL_LIBS})
    endif()
    pl_add_feature("GL_PROC_ADDR" ON) # Defaulting to auto/on logic
else()
    pl_add_feature("OPENGL" OFF)
    pl_add_feature("GL_PROC_ADDR" OFF)
endif()

# Shaderc / Glslang
# Simplified for migration: assume internal or system.
if(PL_USE_SHADERC)
    find_path(SHADERC_INCLUDE_DIR shaderc/shaderc.h)
    find_library(SHADERC_LIBRARY shaderc_shared)
    if(NOT SHADERC_LIBRARY)
        find_library(SHADERC_LIBRARY shaderc)
    endif()

    if(SHADERC_INCLUDE_DIR AND SHADERC_LIBRARY)
         list(APPEND PL_DEPS ${SHADERC_LIBRARY})
         list(APPEND PL_INC_DIRS ${SHADERC_INCLUDE_DIR})
         pl_add_feature("SHADERC" ON)
         # Define internal flags about versions if needed
         # set(PL_HAVE_SHADERC_VK_1_3 1)
    else()
         pl_add_feature("SHADERC" OFF)
    endif()
else()
    pl_add_feature("SHADERC" OFF)
endif()


# DOVI
if(PL_USE_DOVI)
    pl_add_feature("DOVI" ON)
else()
    pl_add_feature("DOVI" OFF)
endif()
if(PL_USE_LIBDOVI)
    find_path(LIBDOVI_INCLUDE_DIR libdovi/rpu_parser.h)
    find_library(LIBDOVI_LIBRARY dovi)
    if(LIBDOVI_INCLUDE_DIR AND LIBDOVI_LIBRARY)
        list(APPEND PL_DEPS ${LIBDOVI_LIBRARY})
        list(APPEND PL_INC_DIRS ${LIBDOVI_INCLUDE_DIR})
        pl_add_feature("LIBDOVI" ON)
    else()
        pl_add_feature("LIBDOVI" OFF)
    endif()
else()
    pl_add_feature("LIBDOVI" OFF)
endif()


# --- Library Target ---
# Defined early so subdirectories can target_sources()
add_library(placebo ${PL_SOURCES}) # Base sources

target_include_directories(placebo PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_include_directories(placebo PRIVATE
    ${PL_INC_DIRS}
)
target_link_libraries(placebo PRIVATE ${PL_DEPS})

# Add preprocessed sources now that the target exists
pl_glsl_preproc("renderer.c" "." placebo)
pl_glsl_preproc("shaders.c" "." placebo)

# --- Subdirectories ---
# src/glsl
add_subdirectory(glsl)

# src/opengl
add_subdirectory(opengl)

# src/shaders
add_subdirectory(shaders)

# src/include/libplacebo (Config Generation)
# Variables needed: majorver, apiver, extra_defs
set(majorver ${PL_MAJOR_VER})
set(apiver ${PL_API_VER})
set(extra_defs ${PL_EXTRA_DEFS})
add_subdirectory(include/libplacebo)

# config_internal.h
# Needs: BUILD_API_VER, BUILD_FIX_VER, PL_HAVE_UNWIND, etc.
set(BUILD_API_VER ${PL_API_VER})
set(BUILD_FIX_VER ${PL_FIX_VER})

# I will create config_internal.h.in content dynamically or use file(WRITE).
# But configure_file is cleaner. I'll write a temporary .in file or just file(GENERATE).
# But wait, I don't know all the internal defines needed.
# From meson.build: PL_DEBUG_ABORT, PL_HAVE_DBGHELP, PL_HAVE_UNWIND, PL_HAVE_EXECINFO, PL_HAVE_SHADERC_VK_*
# I'll create a minimal config_internal.h.in for now with just these known ones.

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/config_internal.h.in
"#ifndef PL_CONFIG_INTERNAL_H
#define PL_CONFIG_INTERNAL_H
#define BUILD_API_VER ${PL_API_VER}
#define BUILD_FIX_VER ${PL_FIX_VER}
#cmakedefine01 PL_DEBUG_ABORT
#cmakedefine01 PL_HAVE_UNWIND

// Default to supporting modern Vulkan environments if checking logic is missing
#ifndef PL_HAVE_SHADERC_VK_1_3
#define PL_HAVE_SHADERC_VK_1_3 1
#endif
// Check if shaderc actually supports 1.4 by trying to compile something or just disable it if erroring?
// The previous error suggests shaderc_env_version_vulkan_1_4 is NOT defined in <shaderc/shaderc.h>.
// So we should NOT define PL_HAVE_SHADERC_VK_1_4 if the header doesn't have it.
// But we are in a config header. 
// We should disable PL_HAVE_SHADERC_VK_1_4 by default if we can't detect it, or handle it in code.
// #cmakedefine01 PL_HAVE_SHADERC_VK_1_4
#undef PL_HAVE_SHADERC_VK_1_4

// #cmakedefine01 PL_HAVE_SHADERC_VK_1_3
// #cmakedefine01 PL_HAVE_SHADERC_VK_1_4
#endif
"
)
configure_file(${CMAKE_CURRENT_BINARY_DIR}/config_internal.h.in ${CMAKE_CURRENT_BINARY_DIR}/config_internal.h)


# Windows RC
if(WIN32)
    set(PL_MAJOR ${PL_MAJOR_VER})
    set(PL_MINOR ${PL_API_VER})
    set(PL_PATCH ${PL_FIX_VER})
    configure_file(
        ${CMAKE_SOURCE_DIR}/win32/libplacebo.rc.in
        ${CMAKE_CURRENT_BINARY_DIR}/libplacebo.rc
    )
    target_sources(placebo PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/libplacebo.rc)
    target_include_directories(placebo PRIVATE "${CMAKE_SOURCE_DIR}/win32")
endif()

# --- Library ---
# add_library(placebo ${PL_SOURCES}) # Moved up

# Install
install(TARGETS placebo EXPORT libplaceboTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include
)
install(DIRECTORY include/libplacebo DESTINATION include)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/include/libplacebo/config.h DESTINATION include/libplacebo)

add_custom_command(
        OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/version.h
        COMMAND ${Python3_EXECUTABLE} "${CMAKE_CURRENT_SOURCE_DIR}/version.py"
                "${CMAKE_CURRENT_SOURCE_DIR}/version.h.in"
                "${CMAKE_CURRENT_BINARY_DIR}/version.h"
                "${CMAKE_SOURCE_DIR}"
                "${PL_VERSION_PRETTY}"
        DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/version.h.in"
        COMMENT "Generating version.h"
    )
    target_sources(placebo PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/version.h)

    # --- Subdirectories ---

