# src/opengl/CMakeLists.txt

if(PL_HAVE_OPENGL)
    # Define OpenGL Extensions
    set(GL_EXTENSIONS
        GL_AMD_pinned_memory
        GL_ARB_buffer_storage
        GL_ARB_compute_shader
        GL_ARB_framebuffer_object
        GL_ARB_get_program_binary
        GL_ARB_invalidate_subdata
        GL_ARB_pixel_buffer_object
        GL_ARB_program_interface_query
        GL_ARB_shader_image_load_store
        GL_ARB_shader_storage_buffer_object
        GL_ARB_sync
        GL_ARB_texture_float
        GL_ARB_texture_gather
        GL_ARB_texture_rg
        GL_ARB_timer_query
        GL_ARB_uniform_buffer_object
        GL_ARB_vertex_array_object
        GL_ARB_half_float_pixel
        GL_EXT_EGL_image_storage
        GL_EXT_color_buffer_float
        GL_EXT_texture3D
        GL_EXT_texture_format_BGRA8888
        GL_EXT_texture_norm16
        GL_EXT_texture_rg
        GL_EXT_unpack_subimage
        GL_KHR_debug
        GL_OES_EGL_image
        GL_OES_EGL_image_external
        EGL_EXT_image_dma_buf_import
        EGL_EXT_image_dma_buf_import_modifiers
        EGL_EXT_platform_base
        EGL_KHR_debug
        EGL_KHR_image_base
        EGL_MESA_image_dma_buf_export
        EGL_MESA_platform_surfaceless
    )
    string(REPLACE ";" "," GL_EXTENSIONS_STR "${GL_EXTENSIONS}")

    set(GLAD_OUT_DIR "${CMAKE_CURRENT_BINARY_DIR}") # Output root for glad
    # file(MAKE_DIRECTORY ${GLAD_OUT_DIR}) # Implicit

    set(GLAD_ARGS
        --out-path=${GLAD_OUT_DIR}
        --reproducible
        --merge
        --api=gl:core,gles2,egl
        --extensions=${GL_EXTENSIONS_STR}
        c
        --header-only
        --mx
    )

    if(PL_HAVE_GL_PROC_ADDR)
        list(APPEND GLAD_ARGS --loader)
    endif()
    
    # We need to use the python path from parent scope
    # Ensure it's passed or available. Subdir inherits vars.
    
    add_custom_command(
        OUTPUT ${GLAD_OUT_DIR}/include/glad/gl.h
        COMMAND ${CMAKE_COMMAND} -E env "PYTHONPATH=${PYTHON_PATH_STR}"
                ${Python3_EXECUTABLE} -m glad ${GLAD_ARGS}
        COMMENT "Generating glad headers"
    )
    
    # Create a custom target for dependency handling
    add_custom_target(generate_glad DEPENDS ${GLAD_OUT_DIR}/include/glad/gl.h)
    
    # Create OBJECT library to handle dependencies correctly within this directory scope
    set(OPENGL_SOURCES
        context.c
        formats.c
        loader_gl.c
        loader_egl.c
        gpu.c
        gpu_pass.c
        gpu_tex.c
        # gpu_buf.c
        swapchain.c
        utils.c
    )
    
    # Explicitly enforce dependency for all sources to ensure header exists before compilation
    set_source_files_properties(${OPENGL_SOURCES} PROPERTIES OBJECT_DEPENDS ${GLAD_OUT_DIR}/include/glad/gl.h)

    add_library(placebo_opengl OBJECT
        ${OPENGL_SOURCES}
        ${GLAD_OUT_DIR}/include/glad/gl.h
    )
    
    # Add dependency on the glad generation target
    add_dependencies(placebo_opengl generate_glad)
    
    target_include_directories(placebo_opengl PRIVATE 
        ${GLAD_OUT_DIR}/include
        ${CMAKE_CURRENT_SOURCE_DIR}/../include 
        ${CMAKE_CURRENT_SOURCE_DIR}/..
        ${CMAKE_CURRENT_BINARY_DIR}/../include # for config.h
    )
    # Inherit other includes if checking finding headers
    target_include_directories(placebo_opengl PRIVATE ${PL_INC_DIRS})

    # Add to main target
    target_sources(placebo PRIVATE $<TARGET_OBJECTS:placebo_opengl>)
    # Flatten includes to main target as well so it can see glad if needed elsewhere (unlikely but safe)
    target_include_directories(placebo PRIVATE ${GLAD_OUT_DIR}/include)

    
    # Check if gpu_buf.c, gpu_tex.c, gpu_pass.c exist in src/opengl/ (meson had them in list)
    # meson: 'opengl/gpu.c', 'opengl/gpu_tex.c', 'opengl/gpu_pass.c', 'opengl/swapchain.c', 'opengl/utils.c'
    # My previous list in src/opengl/CMakeLists.txt might have been incomplete.
    
    # 3rdparty/glad handling (if not handled globally)
    # The generated headers replace the manual include if any? 
    # Yes, we are generating them now.
endif()
