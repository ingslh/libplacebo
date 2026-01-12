# src/glsl/CMakeLists.txt

# Shaderc support
if(PL_HAVE_SHADERC)
    target_sources(placebo PRIVATE spirv_shaderc.c)
    
    # Internal defines for shaderc versions - rough approximation or check symbols
    # Assuming standard modern shaderc for now as per previous CMake logic
    if(NOT DEFINED PL_HAVE_SHADERC_VK_1_3)
         # Default true or check? Meson checks symbols.
         # For now we rely on config_internal.h generation in parent or here?
         # Parent generates config_internal.h.
    endif()
endif()

# spirv.c is common
target_sources(placebo PRIVATE spirv.c)
