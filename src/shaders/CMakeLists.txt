# src/shaders/CMakeLists.txt

set(SHADER_SOURCES
    colorspace.c
    custom.c
    custom_mpv.c
    deinterlacing.c
    dithering.c
    film_grain.c
    film_grain_av1.c
    film_grain_h274.c
    icc.c
    lut.c
    sampling.c
)

# Create an object library to own the generated sources (scoping rule for add_custom_command)
add_library(placebo_shaders OBJECT)
target_include_directories(placebo_shaders PRIVATE ${PL_INC_DIRS})
set_property(TARGET placebo_shaders PROPERTY POSITION_INDEPENDENT_CODE ON)

foreach(_shader ${SHADER_SOURCES})
    pl_glsl_preproc(${_shader} "." placebo_shaders)
endforeach()

# Add the compiled objects to the main library
target_sources(placebo PRIVATE $<TARGET_OBJECTS:placebo_shaders>)
