# demos/CMakeLists.txt

find_package(PkgConfig QUIET)

# Dependencies
pkg_check_modules(GLFW3 glfw3)
pkg_check_modules(SDL2 sdl2)
pkg_check_modules(SDL2_IMAGE SDL2_image)

pkg_check_modules(FFMPEG libavcodec libavformat libavutil)

# Configuration Data
set(HAVE_NUKLEAR 0)
set(HAVE_EGL 0)
set(DEMO_DEFS "")

# Nuklear
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/nuklear/nuklear.h")
    set(HAVE_NUKLEAR 1)
    list(APPEND DEMO_DEFS "HAVE_NUKLEAR=1")
    add_library(nuklear STATIC ui.c)
    target_include_directories(nuklear PUBLIC 
        "${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/nuklear"
        "${CMAKE_CURRENT_BINARY_DIR}" # For config_demos.h
        "${CMAKE_SOURCE_DIR}/src"     # For internal headers if common.h includes them
    )
    target_link_libraries(nuklear PRIVATE placebo)
    target_compile_options(nuklear PRIVATE "-O2")
    if(NOT MSVC)
        target_compile_options(nuklear PRIVATE "-Wno-missing-prototypes")
    endif()
else()
    list(APPEND DEMO_DEFS "HAVE_NUKLEAR=0")
endif()

# Check EGL
# find_path(EGL_INCLUDE_DIR EGL/egl.h) ... simplifying
if(PL_USE_OPENGL) # Approximation logic
    list(APPEND DEMO_DEFS "HAVE_EGL=1")
endif()

# Objects for window backends to handle distinct compile flags
set(BACKEND_OBJECTS "")
set(BACKEND_LIBS "")
set(BACKEND_INCS "")

if(GLFW3_FOUND)
    list(APPEND BACKEND_INCS ${GLFW3_INCLUDE_DIRS})
    list(APPEND BACKEND_LIBS ${GLFW3_LIBRARIES})

    if(PL_USE_VULKAN)
        list(APPEND DEMO_DEFS "HAVE_GLFW_VULKAN=1")
        add_library(glfw_vk OBJECT window_glfw.c)
        target_compile_definitions(glfw_vk PRIVATE USE_VK ${DEMO_DEFS})
        target_include_directories(glfw_vk PRIVATE 
            ${GLFW3_INCLUDE_DIRS} 
            ${CMAKE_CURRENT_SOURCE_DIR} 
            ${CMAKE_CURRENT_BINARY_DIR} # Find config_demos.h
            ${CMAKE_SOURCE_DIR}/src # For internal headers
        )
        target_link_libraries(glfw_vk PRIVATE placebo) # For headers
        list(APPEND BACKEND_OBJECTS $<TARGET_OBJECTS:glfw_vk>)
    endif()

    if(PL_USE_OPENGL)
        list(APPEND DEMO_DEFS "HAVE_GLFW_OPENGL=1")
        add_library(glfw_gl OBJECT window_glfw.c)
        target_compile_definitions(glfw_gl PRIVATE USE_GL ${DEMO_DEFS})
        target_include_directories(glfw_gl PRIVATE 
            ${GLFW3_INCLUDE_DIRS} 
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${CMAKE_CURRENT_BINARY_DIR} # Find config_demos.h
            ${CMAKE_SOURCE_DIR}/src # For internal headers
        )
        target_link_libraries(glfw_gl PRIVATE placebo)
        list(APPEND BACKEND_OBJECTS $<TARGET_OBJECTS:glfw_gl>)
    endif()
    
    if(PL_USE_D3D11 AND WIN32) # Check WIN32 to prevent building on Linux
        list(APPEND DEMO_DEFS "HAVE_GLFW_D3D11=1")
        add_library(glfw_d3d11 OBJECT window_glfw.c)
        target_compile_definitions(glfw_d3d11 PRIVATE USE_D3D11 ${DEMO_DEFS})
        target_include_directories(glfw_d3d11 PRIVATE 
            ${GLFW3_INCLUDE_DIRS} 
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${CMAKE_CURRENT_BINARY_DIR} # Find config_demos.h
            ${CMAKE_SOURCE_DIR}/src # For internal headers
        )
         target_link_libraries(glfw_d3d11 PRIVATE placebo)
        list(APPEND BACKEND_OBJECTS $<TARGET_OBJECTS:glfw_d3d11>)
    endif()
endif()

if(SDL2_FOUND)
    list(APPEND BACKEND_INCS ${SDL2_INCLUDE_DIRS})
    list(APPEND BACKEND_LIBS ${SDL2_LIBRARIES})
    
    if(PL_USE_VULKAN)
        list(APPEND DEMO_DEFS "HAVE_SDL_VULKAN=1")
        add_library(sdl_vk OBJECT window_sdl.c)
        target_compile_definitions(sdl_vk PRIVATE USE_VK ${DEMO_DEFS})
        target_include_directories(sdl_vk PRIVATE 
            ${SDL2_INCLUDE_DIRS} 
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${CMAKE_CURRENT_BINARY_DIR} # Find config_demos.h
            ${CMAKE_SOURCE_DIR}/src # For internal headers
        )
        target_link_libraries(sdl_vk PRIVATE placebo)
        list(APPEND BACKEND_OBJECTS $<TARGET_OBJECTS:sdl_vk>)
    endif()
    
    if(PL_USE_OPENGL)
        list(APPEND DEMO_DEFS "HAVE_SDL_OPENGL=1")
        add_library(sdl_gl OBJECT window_sdl.c)
        target_compile_definitions(sdl_gl PRIVATE USE_GL ${DEMO_DEFS})
        target_include_directories(sdl_gl PRIVATE 
            ${SDL2_INCLUDE_DIRS} 
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${CMAKE_CURRENT_BINARY_DIR} # Find config_demos.h
            ${CMAKE_SOURCE_DIR}/src # For internal headers
        )
        target_link_libraries(sdl_gl PRIVATE placebo)
        list(APPEND BACKEND_OBJECTS $<TARGET_OBJECTS:sdl_gl>)
    endif()
endif()


# Generate config_demos.h
# We'll just pass defines via compile definitions for simplicity or generate it.
# The code probably includes "config_demos.h".
# Let's generate it.
set(CONFIG_DEMOS_CONTENT "#ifndef CONFIG_DEMOS_H\n#define CONFIG_DEMOS_H\n")
foreach(_def ${DEMO_DEFS})
    # _def is KEY=VAL or KEY
    string(REPLACE "=" " " _def_composed ${_def})
    string(APPEND CONFIG_DEMOS_CONTENT "#define ${_def_composed}\n")
endforeach()
string(APPEND CONFIG_DEMOS_CONTENT "#endif\n")
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/config_demos.h ${CONFIG_DEMOS_CONTENT})


# Demo Common Lib
# If no backends, we warn but here we just proceed.
add_library(demo_common STATIC window.c utils.c ${BACKEND_OBJECTS})
target_include_directories(demo_common PUBLIC 
    ${CMAKE_CURRENT_SOURCE_DIR} 
    ${CMAKE_CURRENT_BINARY_DIR} 
    ${BACKEND_INCS}
    ${CMAKE_SOURCE_DIR}/src # For internal headers like pl_clock.h
)
target_link_libraries(demo_common PUBLIC placebo ${BACKEND_LIBS})

# Add internal source dir to executables who might need internals directly
target_include_directories(demo_common PUBLIC ${CMAKE_SOURCE_DIR}/src)
if(WIN32)
     target_link_libraries(demo_common PUBLIC winmm)
endif()

# Executables
add_executable(colors colors.c)
target_link_libraries(colors PRIVATE demo_common)
install(TARGETS colors RUNTIME DESTINATION bin)

if(SDL2_IMAGE_FOUND)
    add_executable(sdlimage sdlimage.c)
    target_link_libraries(sdlimage PRIVATE demo_common ${SDL2_IMAGE_LIBRARIES})
    target_include_directories(sdlimage PRIVATE ${SDL2_IMAGE_INCLUDE_DIRS})
    install(TARGETS sdlimage RUNTIME DESTINATION bin)
endif()

if(FFMPEG_FOUND)
    set(PLPLAY_SRCS plplay.c settings.c)
    if(WIN32)
        # Handle Windows RC
        # configure_file demos.rc
        configure_file(
            ${CMAKE_SOURCE_DIR}/win32/demos.rc.in
            ${CMAKE_CURRENT_BINARY_DIR}/demos.rc
        )
        list(APPEND PLPLAY_SRCS ${CMAKE_CURRENT_BINARY_DIR}/demos.rc)
    endif()

    add_executable(plplay ${PLPLAY_SRCS})
    target_link_libraries(plplay PRIVATE demo_common ${FFMPEG_LIBRARIES})
    target_include_directories(plplay PRIVATE ${FFMPEG_INCLUDE_DIRS})
    if(HAVE_NUKLEAR)
        target_link_libraries(plplay PRIVATE nuklear)
    endif()
    if(WIN32)
        target_link_libraries(plplay PRIVATE shlwapi)
    endif()
    install(TARGETS plplay RUNTIME DESTINATION bin)
endif()

# Headless check (simple approx)
if(PL_USE_VULKAN)
    add_executable(video-filtering video-filtering.c)
    target_link_libraries(video-filtering PRIVATE placebo demo_common) # Need demo_common or config_demos.h includes
    target_include_directories(video-filtering PRIVATE 
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_SOURCE_DIR}/src
    )
    install(TARGETS video-filtering RUNTIME DESTINATION bin)
    
    add_executable(multigpu-bench multigpu-bench.c)
    target_link_libraries(multigpu-bench PRIVATE placebo demo_common) # Need demo_common or config_demos.h includes
    target_include_directories(multigpu-bench PRIVATE 
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_SOURCE_DIR}/src
    )
    install(TARGETS multigpu-bench RUNTIME DESTINATION bin)
endif()
