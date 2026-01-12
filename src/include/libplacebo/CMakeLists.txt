# src/include/libplacebo/CMakeLists.txt

# Generate config.h
# Requires: majorver, apiver, extra_defs to be defined in parent scope
configure_file(
    config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/config.h
    @ONLY
)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/config.h DESTINATION include/libplacebo)

# Add binary dir to include path so code can find config.h
target_include_directories(placebo PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
)
