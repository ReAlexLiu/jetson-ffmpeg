cmake_minimum_required(VERSION 3.9)
project(nvmpi VERSION 1.0.0 DESCRIPTION "nvidia multimedia api")

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif ()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED on)

set(OPT_FLAGS "-Ofast -march=native")
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OPT_FLAGS}")
#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OPT_FLAGS}")

set(MMAPI_PATH /usr/src/jetson_multimedia_api/)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Wno-unused-value -fPIC")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wno-unused-value -fPIC")

set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-rpath-link=/usr/lib/aarch64-linux-gnu/tegra")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-rpath-link=/usr/local/cuda/lib64")

find_package(Threads)

set(NVMPI_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(NVMPI_PUBLIC_HEADER ${NVMPI_INCLUDE_DIR}/nvmpi.h)
set(CUDA_INCLUDE_DIR /usr/local/cuda/include)
set(JETSON_MULTIMEDIA_API_INCLUDE_DIR ${MMAPI_PATH}/include)
set(JETSON_MULTIMEDIA_API_SRC_DIR ${MMAPI_PATH}/samples/common/classes)

set(NVMPI_INCLUDE_DIRS
        ${NVMPI_INCLUDE_DIR}
        ${CUDA_INCLUDE_DIR}
        ${JETSON_MULTIMEDIA_API_INCLUDE_DIR})

file(GLOB MAIN_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/src/nvmpi_dec.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/src/nvmpi_enc.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/src/NVMPI_frameBuf.cpp)

#if NvUtils API is present prefer it to nvbuf_utils
if (EXISTS "${MMAPI_PATH}/include/nvbufsurface.h")
    add_definitions(-DWITH_NVUTILS)

    file(GLOB MMAPI_SRC
            ${MMAPI_PATH}/samples/common/classes/NvBuffer.cpp
            ${MMAPI_PATH}/samples/common/classes/NvElement.cpp
            ${MMAPI_PATH}/samples/common/classes/NvElementProfiler.cpp
            ${MMAPI_PATH}/samples/common/classes/NvLogging.cpp
            ${MMAPI_PATH}/samples/common/classes/NvV4l2Element.cpp
            ${MMAPI_PATH}/samples/common/classes/NvV4l2ElementPlane.cpp
            ${MMAPI_PATH}/samples/common/classes/NvVideoDecoder.cpp
            ${MMAPI_PATH}/samples/common/classes/NvVideoEncoder.cpp
            ${MMAPI_PATH}/samples/common/classes/NvBufSurface.cpp)

    add_library(nvmpi SHARED ${MAIN_SRC} ${MMAPI_SRC})
    add_library(nvmpi_static STATIC ${MAIN_SRC} ${MMAPI_SRC})

    set(LIBS nvv4l2 nvbufsurface nvbufsurftransform)
    foreach (LIB ${LIBS})
        find_library(LIB_${LIB} ${LIB} PATHS /usr/lib/aarch64-linux-gnu/tegra)
        if (LIB_${LIB})
            target_link_libraries(nvmpi PRIVATE ${LIB_${LIB}})
            target_link_libraries(nvmpi_static PRIVATE ${LIB_${LIB}})
        else ()
            message("Could not find LIB_${LIB}")
        endif ()
    endforeach ()
    set(NVMPI_SRC ${NVMPI_SRC}
            ${MMAPI_PATH}/samples/common/classes/NvBufSurface.cpp)
    set(NVMPI_DEP_LIBS ${NVMPI_DEP_LIBS} ${LIB_NVBUFSURFACE} ${LIB_NVBUFSURFTRANSFORM})
else ()
    file(GLOB NVMPI_SRC
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvBuffer.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvElement.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvElementProfiler.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvLogging.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvV4l2Element.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvV4l2ElementPlane.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvVideoDecoder.cpp
            ${JETSON_MULTIMEDIA_API_SRC_DIR}/NvVideoEncoder.cpp)

    find_library(LIB_NVBUF nvbuf_utils PATHS /usr/lib/aarch64-linux-gnu/tegra)
    find_library(LIB_V4L2 nvv4l2 PATHS /usr/lib/aarch64-linux-gnu/tegra)
    find_library(LIB_NVJPEG nvjpeg PATHS /usr/lib/aarch64-linux-gnu/tegra)
    set(NVMPI_DEP_LIBS ${CMAKE_THREAD_LIBS_INIT} ${LIB_NVBUF} ${LIB_V4L2} ${LIB_NVJPEG})
    
    add_library(nvmpi SHARED ${MAIN_SRC} ${NVMPI_SRC})
    add_library(nvmpi_static ${MAIN_SRC} STATIC ${NVMPI_SRC})
endif ()

set_target_properties(nvmpi_static PROPERTIES OUTPUT_NAME nvmpi)
set_target_properties(nvmpi PROPERTIES VERSION ${PROJECT_VERSION})
set_target_properties(nvmpi PROPERTIES SOVERSION 1)
set_target_properties(nvmpi nvmpi_static PROPERTIES PUBLIC_HEADER ${NVMPI_PUBLIC_HEADER})
set_target_properties(nvmpi PROPERTIES LINK_FLAGS "-Wl,--no-as-needed")
target_link_libraries(nvmpi PRIVATE ${NVMPI_DEP_LIBS})
target_include_directories(nvmpi PRIVATE ${NVMPI_INCLUDE_DIRS})
target_include_directories(nvmpi_static PRIVATE ${NVMPI_INCLUDE_DIRS})
configure_file(nvmpi.pc.in nvmpi.pc @ONLY)
include(GNUInstallDirs)
install(TARGETS nvmpi nvmpi_static
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES ${CMAKE_BINARY_DIR}/nvmpi.pc
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/pkgconfig)
install(FILES ${CMAKE_BINARY_DIR}/nvmpi.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
