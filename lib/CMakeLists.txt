project(libsegyio C)

set(SOURCE_FILES src/segy.c)

set(STATIC_NAME segyio)
if(MSVC)
    # MSVC outputs the same name for static and shared libraries (with the same
    # extension), so we need to differentiate between the two somehow.
    set(STATIC_NAME "${STATIC_NAME}-static")
    set(DLL_EXPORT_FILES src/segy.def)
endif()

#
# static build
#
add_library(segyio-static STATIC ${SOURCE_FILES})
target_link_libraries(segyio-static ${m} ${ws2})
target_compile_definitions(segyio-static PRIVATE
    ${htons} ${mmap} ${fstat} ${ftello})
target_compile_options(segyio-static BEFORE
    PRIVATE $<$<CONFIG:Debug>:${warnings-c}> ${c99}
)
set_target_properties(segyio-static PROPERTIES
                      OUTPUT_NAME ${STATIC_NAME}
                      CLEAN_DIRECT_OUTPUT 1)
target_include_directories(
    segyio-static PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    PRIVATE src include)

#
# dynamic build
#
add_library(segyio-shared SHARED ${SOURCE_FILES} ${DLL_EXPORT_FILES})
target_link_libraries(segyio-shared ${m} ${ws2})
target_compile_options(segyio-shared BEFORE
    PRIVATE $<$<CONFIG:Debug>:${warnings-c}> ${c99}
)
target_compile_definitions(segyio-shared PRIVATE
    ${htons} ${mmap} ${fstat} ${ftello})
set_target_properties(segyio-shared PROPERTIES
                      SOVERSION   ${segyio_MAJOR}
                      VERSION     ${segyio_MAJOR}
                      OUTPUT_NAME segyio
                      CLEAN_DIRECT_OUTPUT 1)
target_include_directories(
    segyio-shared PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    PRIVATE src include)

if (BUILD_SHARED_LIBS)
    add_library(segyio ALIAS segyio-shared)
else ()
    add_library(segyio ALIAS segyio-static)
    set_property(TARGET segyio-static PROPERTY POSITION_INDEPENDENT_CODE ON)
endif ()

#
# install & export
#
install(TARGETS segyio-static segyio-shared
        EXPORT segyio
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(DIRECTORY include/ DESTINATION include)
install(EXPORT segyio DESTINATION share/segyio/cmake FILE segyio-config.cmake)
export(TARGETS segyio-static segyio-shared FILE segyio-config.cmake)

if (NOT BUILD_TESTING)
    return ()
endif ()

configure_file(${testdata}/small.sgy test-data/small.sgy             COPYONLY)
configure_file(${testdata}/text.sgy  test-data/text.sgy              COPYONLY)

add_executable(c.segy test/testsuite.cpp
                      test/segy.cpp
                      test/mmap.cpp)
target_include_directories(c.segy PRIVATE src)
target_link_libraries(c.segy catch2 segyio)
target_compile_options(c.segy BEFORE PRIVATE
                       ${c++11} $<$<CONFIG:Debug>:${warnings-c}> ${mmap})
add_test(NAME c.segy COMMAND c.segy ~[mmap] [c.segy])
add_test(NAME c.segy.mmap COMMAND c.segy [mmap])
