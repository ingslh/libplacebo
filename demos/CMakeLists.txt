# demos/CMakeLists.txt

# Dependencies

# GLFW3
find_package(glfw3 CONFIG)
if(NOT glfw3_FOUND)
    find_path(GLFW3_INCLUDE_DIRS GLFW/glfw3.h)
    find_library(GLFW3_LIBRARIES glfw3 glfw)
    if(GLFW3_INCLUDE_DIRS AND GLFW3_LIBRARIES)
        set(GLFW3_FOUND TRUE)
    endif()
endif()

# SDL2
find_package(SDL2 CONFIG)
if(NOT SDL2_FOUND)
    find_path(SDL2_INCLUDE_DIRS SDL.h PATH_SUFFIXES SDL2)
    find_library(SDL2_LIBRARIES SDL2)
    if(SDL2_INCLUDE_DIRS AND SDL2_LIBRARIES)
        set(SDL2_FOUND TRUE)
    endif()
endif()

# SDL2_image
# Simplified check
find_path(SDL2_IMAGE_INCLUDE_DIRS SDL_image.h PATH_SUFFIXES SDL2)
find_library(SDL2_IMAGE_LIBRARIES SDL2_image)
if(SDL2_IMAGE_INCLUDE_DIRS AND SDL2_IMAGE_LIBRARIES)
    set(SDL2_IMAGE_FOUND TRUE)
endif()

# FFMPEG
find_path(AVCODEC_INCLUDE_DIR libavcodec/avcodec.h)
find_library(AVCODEC_LIBRARY avcodec)
find_path(AVFORMAT_INCLUDE_DIR libavformat/avformat.h)
find_library(AVFORMAT_LIBRARY avformat)
find_path(AVUTIL_INCLUDE_DIR libavutil/avutil.h)
find_library(AVUTIL_LIBRARY avutil)

if(AVCODEC_INCLUDE_DIR AND AVCODEC_LIBRARY AND AVFORMAT_LIBRARY AND AVUTIL_LIBRARY)
    set(FFMPEG_FOUND TRUE)
    set(FFMPEG_INCLUDE_DIRS ${AVCODEC_INCLUDE_DIR} ${AVFORMAT_INCLUDE_DIR} ${AVUTIL_INCLUDE_DIR})
    set(FFMPEG_LIBRARIES ${AVCODEC_LIBRARY} ${AVFORMAT_LIBRARY} ${AVUTIL_LIBRARY})
endif()


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
find_path(EGL_INCLUDE_DIR EGL/egl.h)
if(PL_USE_OPENGL AND EGL_INCLUDE_DIR)
    list(APPEND DEMO_DEFS "HAVE_EGL=1")
    include_directories(${EGL_INCLUDE_DIR})
endif()

# Vulkan headers
if(PL_USE_VULKAN)
    if(EXISTS "${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/include")
        include_directories("${CMAKE_SOURCE_DIR}/3rdparty/Vulkan-Headers/include")
    endif()
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
    target_link_libraries(video-filtering PRIVATE demo_common) # Need demo_common or config_demos.h includes
    target_include_directories(video-filtering PRIVATE 
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_SOURCE_DIR}/src
    )
    install(TARGETS video-filtering RUNTIME DESTINATION bin)
    
    add_executable(multigpu-bench multigpu-bench.c)
    target_link_libraries(multigpu-bench PRIVATE demo_common) # Need demo_common or config_demos.h includes
    target_include_directories(multigpu-bench PRIVATE 
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_SOURCE_DIR}/src
    )
    install(TARGETS multigpu-bench RUNTIME DESTINATION bin)
endif()
