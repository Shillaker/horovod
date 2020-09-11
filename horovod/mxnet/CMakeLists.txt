if("$ENV{HOROVOD_WITHOUT_MXNET}" STREQUAL "1")
    return()
endif()

set(Mxnet_TARGET_LIB "mxnet")

# Find MXNet
if(HOROVOD_FAASM)
    message(STATUS "Using Faasm MXNet lib")
    set(MXNET_FOUND TRUE)

    set(Mxnet_INCLUDE_DIRS 
        ${CMAKE_SYSROOT}/include/mxnet
        )

    set(Mxnet_LIBRARIES 
        ${CMAKE_SYSROOT}/lib/wasm32-wasi/libmxnet.a
        ${CMAKE_SYSROOT}/lib/wasm32-wasi/libdmlc.a
        )
else()
    set(Mxnet_REQUIRED "")
    if ("$ENV{HOROVOD_WITH_MXNET}" STREQUAL "1")
        set(Mxnet_REQUIRED "REQUIRED")
    endif ()
    find_package(Mxnet "1.4.0" ${Mxnet_REQUIRED})
endif()

if(NOT MXNET_FOUND)
    return()
endif()

if (HAVE_CUDA AND NOT Mxnet_USE_CUDA)
    message(FATAL_ERROR "Horovod build with GPU support was requested but this MXNet installation does not support CUDA.")
elseif (Mxnet_USE_CUDA AND NOT HAVE_CUDA)
    add_cuda()
endif()
include_directories(SYSTEM ${Mxnet_INCLUDE_DIRS})
list(APPEND Mxnet_LINKER_LIBS ${Mxnet_LIBRARIES})
if(HAVE_GLOO)
    list(APPEND Mxnet_LINKER_LIBS gloo)
endif()
if(HAVE_CUDA)
    list(APPEND Mxnet_LINKER_LIBS horovod_cuda_kernels)
endif()
set(CMAKE_CXX_FLAGS "${Mxnet_COMPILE_FLAGS} ${CMAKE_CXX_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DMSHADOW_USE_MKL=0 -DMSHADOW_USE_F16C=0")

# MXNet SOURCES
list(APPEND Mxnet_SOURCES "${PROJECT_SOURCE_DIR}/horovod/mxnet/mpi_ops.cc"
                          "${PROJECT_SOURCE_DIR}/horovod/mxnet/tensor_util.cc"
                          "${PROJECT_SOURCE_DIR}/horovod/mxnet/cuda_util.cc"
                          "${PROJECT_SOURCE_DIR}/horovod/mxnet/adapter.cc")

# Create library
set_output_dir()

# When building for Faasm, we need to have a static lib
if(HOROVOD_FAASM)
    add_library(${Mxnet_TARGET_LIB} STATIC ${SOURCES} ${Mxnet_SOURCES})
else()
    add_library(${Mxnet_TARGET_LIB} SHARED ${SOURCES} ${Mxnet_SOURCES})
    set_target_properties(${Mxnet_TARGET_LIB} PROPERTIES SUFFIX "${Python_SUFFIX}")
endif()

target_link_libraries(${Mxnet_TARGET_LIB} ${LINKER_LIBS} ${Mxnet_LINKER_LIBS})
set_target_properties(${Mxnet_TARGET_LIB} PROPERTIES PREFIX "")

# NOTE - here we're setting the name for the final library
set_target_properties(${Mxnet_TARGET_LIB} PROPERTIES OUTPUT_NAME "horovod_mxnet")

# Set up install
install(TARGETS ${Mxnet_TARGET_LIB}
        ARCHIVE DESTINATION ${CMAKE_SYSROOT}/lib/wasm32-wasi
        LIBRARY DESTINATION ${CMAKE_SYSROOT}/lib/wasm32-wasi
        PUBLIC_HEADER DESTINATION ${CMAKE_SYSROOT}/include/horovod
    )

