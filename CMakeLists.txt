cmake_minimum_required(VERSION 3.15...3.26)
project(${SKBUILD_PROJECT_NAME} LANGUAGES CXX VERSION ${SKBUILD_PROJECT_VERSION})

message(STATUS "SKBUILD_PLATLIB_DIR: ${SKBUILD_PLATLIB_DIR}")
message(STATUS "SKBUILD_PROJECT_VERSION: ${SKBUILD_PROJECT_VERSION}")  # 应输出 1.11.0
message(STATUS "CMAKE_PROJECT_VERSION: ${CMAKE_PROJECT_VERSION}")      # 应输出 1.11.0
message(STATUS "PROJECT_SOURCE_DIR: ${PROJECT_SOURCE_DIR}")
message(STATUS "faiss_SOURCE_DIR: ${faiss_SOURCE_DIR}")
message(STATUS "FAISS_ENABLE_GPU: ${FAISS_ENABLE_GPU}")

set(faiss_SOURCE_DIR ${PROJECT_SOURCE_DIR}/faiss)
set(CMAKE_SWIG_OUTDIR ${SKBUILD_PLATLIB_DIR}/faiss)
set(CMAKE_INSTALL_RPATH $ORIGIN)


add_subdirectory(faiss) # 核心库目录，定义了 add_library(faiss ...)
# add_subdirectory(faiss/faiss/python)

install(TARGETS faiss DESTINATION faiss)  # 新增此行，安装 libfaiss.so
install(TARGETS swigfaiss DESTINATION faiss)
install(TARGETS faiss_python_callbacks DESTINATION faiss)

if(FAISS_OPT_LEVEL STREQUAL "avx2")
    install(TARGETS swigfaiss_avx2 DESTINATION faiss)
elseif(FAISS_OPT_LEVEL STREQUAL "avx512")
    install(TARGETS swigfaiss_avx512 DESTINATION faiss)
elseif(FAISS_OPT_LEVEL STREQUAL "sve")
    install(TARGETS swigfaiss_sve DESTINATION faiss)
endif()

configure_file(faiss/faiss/python/__init__.py ${SKBUILD_PLATLIB_DIR}/faiss/__init__.py COPYONLY)
configure_file(faiss/faiss/python/class_wrappers.py ${SKBUILD_PLATLIB_DIR}/faiss/class_wrappers.py COPYONLY)
configure_file(faiss/faiss/python/gpu_wrappers.py ${SKBUILD_PLATLIB_DIR}/faiss/gpu_wrappers.py COPYONLY)
configure_file(faiss/faiss/python/extra_wrappers.py ${SKBUILD_PLATLIB_DIR}/faiss/extra_wrappers.py COPYONLY)
configure_file(faiss/faiss/python/array_conversions.py ${SKBUILD_PLATLIB_DIR}/faiss/array_conversions.py COPYONLY)
file(COPY faiss/contrib DESTINATION ${SKBUILD_PLATLIB_DIR}/faiss/)
configure_file(faiss/faiss/python/loader.py ${SKBUILD_PLATLIB_DIR}/faiss/_loader.py COPYONLY)  
configure_file(patch/loader.py ${SKBUILD_PLATLIB_DIR}/faiss/loader.py COPYONLY)

if (FAISS_ENABLE_GPU)
    if (FAISS_ENABLE_ROCM)
        set(VARIANT "rocm")
    else()
        find_package(CUDAToolkit REQUIRED)
        string(REGEX MATCH "^([0-9]+)" CUDA_MAJOR_VERSION ${CUDAToolkit_VERSION})
        if (FAISS_ENABLE_RAFT)
            set(VARIANT "gpu-raft-cu${CUDA_MAJOR_VERSION}") 
        else()
            set(VARIANT "gpu-cu${CUDA_MAJOR_VERSION}")
        endif()
    endif()
else()
    set(VARIANT "cpu")
endif()

message(STATUS "VARIANT: ${VARIANT}")
configure_file(variant/${VARIANT}/_preload_library.json ${SKBUILD_PLATLIB_DIR}/faiss/_preload_library.json COPYONLY)
