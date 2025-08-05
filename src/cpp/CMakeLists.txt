#include_directories(${CMAKE_SOURCE_DIR}/include algorithms ext util nn .)

add_definitions(-D_FLANN_VERSION=${FLANN_VERSION})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h)

# Instrumentation settings - following liquid-dsp pattern
set(PASS_PATH "$ENV{PASS_PATH}")
set(LLVM_DIR "$ENV{LLVM_DIR}")

if (PASS_PATH AND LLVM_DIR)
    message(STATUS "Instrumentation enabled - PASS_PATH: ${PASS_PATH}")
    set(LLVM_LINK "${LLVM_DIR}/bin/llvm-link")
    message(STATUS "LLVM_LINK: ${LLVM_LINK}")  
    set(OPT "${LLVM_DIR}/bin/opt")
    message(STATUS "OPT: ${OPT}")
    set(CLANG "${LLVM_DIR}/bin/clang")
    message(STATUS "CLANG: ${CLANG}")

    # Set compilers for instrumentation
    set(CMAKE_C_COMPILER clang-18)
    set(CMAKE_CXX_COMPILER clang++)
    set(INSTRUMENTATION_ENABLED TRUE)
else()
    message(STATUS "Instrumentation disabled - PASS_PATH or LLVM_DIR not set")
    set(INSTRUMENTATION_ENABLED FALSE)
endif()

file(GLOB_RECURSE C_SOURCES flann.cpp)
file(GLOB_RECURSE CPP_SOURCES flann_cpp.cpp)
file(GLOB_RECURSE CU_SOURCES *.cu)

if (INSTRUMENTATION_ENABLED)
    # Instrumentation workflow for flann_cpp
    # Step 1: Compile CPP_SOURCES to LLVM IR
foreach(cpp_source ${CPP_SOURCES})
    get_filename_component(source_name ${cpp_source} NAME_WE)
    set(ll_file "${CMAKE_CURRENT_BINARY_DIR}/${source_name}.ll")
    set(ll_instrumented_file "${CMAKE_CURRENT_BINARY_DIR}/${source_name}_instrumented.ll")
    
    # Compile to LLVM IR
    add_custom_command(
        OUTPUT ${ll_file}
        COMMAND ${CLANG} -S -emit-llvm -std=c++17 -fPIC -O2 -DFLANN_USE_LZ4 
                -fno-vectorize -fno-slp-vectorize -ffp-contract=off -mno-avx -mno-avx2 -mno-avx512f -mno-fma
                -I${CMAKE_SOURCE_DIR}/src/cpp -I${CMAKE_SOURCE_DIR}/include 
                -I/home/uvxiao/.local/include -c ${cpp_source} -o ${ll_file}
        DEPENDS ${cpp_source}
    )
    
    # Instrument LLVM IR
    add_custom_command(
        OUTPUT ${ll_instrumented_file}
        COMMAND ${OPT} -load-pass-plugin="${PASS_PATH}" -passes=bb_instrument ${ll_file} -o ${ll_instrumented_file}
        DEPENDS ${ll_file}
    )
    
    list(APPEND FLANN_LL_FILES ${ll_file})
    list(APPEND FLANN_LL_INSTRUMENTED_FILES ${ll_instrumented_file})
endforeach()

# Step 2: Link LLVM IR files into bitcode
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp.bc
    COMMAND ${LLVM_LINK} ${FLANN_LL_FILES} -o ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp.bc
    DEPENDS ${FLANN_LL_FILES}
)

add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc
    COMMAND ${LLVM_LINK} ${FLANN_LL_INSTRUMENTED_FILES} -o ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc
    DEPENDS ${FLANN_LL_INSTRUMENTED_FILES}
)

# Step 3: Create shared library from instrumented bitcode with same name as non-instrumented
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_VERSION}
    COMMAND ${CLANG} -shared -fPIC -fno-vectorize -fno-slp-vectorize -ffp-contract=off -mno-avx -mno-avx2 -mno-avx512f -mno-fma
            ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc 
            -o ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_VERSION}
            -L/home/uvxiao/.local/lib -llz4
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc
)

# Create symlinks for the instrumented library
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_SOVERSION}
    COMMAND ${CMAKE_COMMAND} -E create_symlink libflann_cpp.so.${FLANN_VERSION} ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_SOVERSION}
    COMMAND ${CMAKE_COMMAND} -E create_symlink libflann_cpp.so.${FLANN_SOVERSION} ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_VERSION}
)

# Create custom targets for instrumented artifacts
add_custom_target(flann_cpp_ll ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp.bc)
add_custom_target(flann_cpp_ll_instrumented ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc)
add_custom_target(flann_cpp ALL DEPENDS 
    ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_VERSION}
    ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so
    ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_SOVERSION}
)

    set(flann_install_targets flann_cpp)

else()
    # Non-instrumented build - create regular flann_cpp target
    add_library(flann_cpp SHARED ${CPP_SOURCES})
    target_include_directories(flann_cpp PRIVATE /home/uvxiao/.local/include)
    target_link_directories(flann_cpp PRIVATE /home/uvxiao/.local/lib)
    target_link_libraries(flann_cpp ${LZ4_LINK_LIBRARIES})
    
    # Also create static library for non-instrumented build
    add_library(flann_cpp_s STATIC ${CPP_SOURCES})
    target_include_directories(flann_cpp_s PRIVATE /home/uvxiao/.local/include)
    target_link_directories(flann_cpp_s PRIVATE /home/uvxiao/.local/lib)
    target_link_libraries(flann_cpp_s PUBLIC ${LZ4_LINK_LIBRARIES})
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_CLANG)
        set_target_properties(flann_cpp_s PROPERTIES COMPILE_FLAGS -fPIC)
    endif()
    if (CMAKE_BUILD_STATIC_LIBS)
        list(APPEND flann_install_targets flann_cpp_s)
    else()
        set_target_properties(flann_cpp_s PROPERTIES EXCLUDE_FROM_ALL true)
    endif()

    add_library(flann_cpp SHARED ${CPP_SOURCES})
    target_include_directories(flann_cpp PRIVATE /home/uvxiao/.local/include)
    target_link_directories(flann_cpp PRIVATE /home/uvxiao/.local/lib)
    target_link_libraries(flann_cpp ${LZ4_LINK_LIBRARIES})
    # export lz4 headers, so that MSVC to creates flann_cpp.lib
    set_target_properties(flann_cpp PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS YES)

    set(flann_install_targets flann_cpp)
endif()

if (BUILD_CUDA_LIB)
    SET(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-DFLANN_USE_CUDA;-Xcudafe \"--diag_suppress=partial_override\" ;-gencode=arch=compute_52,code=\"sm_52,compute_52\";-gencode=arch=compute_61,code=\"sm_61,compute_61\"")
    if(CMAKE_COMPILER_IS_GNUCC)
		set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-Xcompiler;-fPIC;" )
        if (NVCC_COMPILER_BINDIR)
            set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};--compiler-bindir=${NVCC_COMPILER_BINDIR}")
        endif()
    else()
	    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};" )
    endif()
    cuda_add_library(flann_cuda_s STATIC ${CU_SOURCES})
    set_property(TARGET flann_cuda_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
    if (CMAKE_BUILD_STATIC_LIBS)
        list(APPEND flann_install_targets flann_cuda_s)
    else()
        set_target_properties(flann_cuda_s PROPERTIES EXCLUDE_FROM_ALL true)
    endif()

    cuda_add_library(flann_cuda SHARED ${CU_SOURCES})
    list(APPEND flann_install_targets flann_cuda)
    if (NOT INSTRUMENTATION_ENABLED)
        set_property(TARGET flann_cpp PROPERTY COMPILE_DEFINITIONS FLANN_USE_CUDA)
        set_property(TARGET flann_cpp_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC FLANN_USE_CUDA)
    endif()
else()
    if (NOT INSTRUMENTATION_ENABLED)
        set_property(TARGET flann_cpp_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
    endif()
endif()

    set_target_properties(flann_cpp PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    )

if (BUILD_CUDA_LIB)
    set_target_properties(flann_cuda PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    )
endif()


if (USE_MPI AND HDF5_IS_PARALLEL)
    add_executable(flann_mpi_server flann/mpi/flann_mpi_server.cpp)
    target_link_libraries(flann_mpi_server flann_cpp ${HDF5_LIBRARIES} ${MPI_LIBRARIES} ${Boost_LIBRARIES})

    add_executable(flann_mpi_client flann/mpi/flann_mpi_client.cpp)
    target_link_libraries(flann_mpi_client flann_cpp ${HDF5_LIBRARIES} ${MPI_LIBRARIES} ${Boost_LIBRARIES})

    install (TARGETS flann_mpi_client flann_mpi_server
        DESTINATION bin)
endif()


if (BUILD_C_BINDINGS)
    add_library(flann_s STATIC ${C_SOURCES})
    target_link_libraries(flann_s ${LZ4_LINK_LIBRARIES})
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_CLANG)
        set_target_properties(flann_s PROPERTIES COMPILE_FLAGS -fPIC)
    endif()
    set_property(TARGET flann_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
    if (CMAKE_BUILD_STATIC_LIBS)
        list(APPEND flann_install_targets flann_s)
    else()
        set_target_properties(flann_s PROPERTIES EXCLUDE_FROM_ALL true)
    endif()

    add_library(flann SHARED ${C_SOURCES})
    target_link_libraries(flann ${LZ4_LINK_LIBRARIES})
    list(APPEND flann_install_targets flann)

    if(MINGW AND OPENMP_FOUND)
        target_link_libraries(flann gomp)
    endif()

    set_target_properties(flann PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    )
endif()

if(WIN32)
if (BUILD_C_BINDINGS AND BUILD_MATLAB_BINDINGS)
    install (
        TARGETS flann
        RUNTIME DESTINATION share/flann/matlab
    )
endif()
endif(WIN32)


if (INSTRUMENTATION_ENABLED)
    # For instrumented build, install custom-generated libraries
    install(FILES 
        ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_VERSION}
        ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so.${FLANN_SOVERSION}
        ${CMAKE_CURRENT_BINARY_DIR}/libflann_cpp.so
        DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
    
    # Install the bitcode files as well
    install(FILES 
        ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp.bc
        ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.bc
        DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )

    # Install the llvm files as well
    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp.ll
        ${CMAKE_CURRENT_BINARY_DIR}/flann_cpp_instrumented.ll
        DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
    
    # Create an interface target that properly represents the instrumented library
    add_library(flann_cpp_export INTERFACE)
    # Set the library properties so it can be found
    set_target_properties(flann_cpp_export PROPERTIES
        INTERFACE_LINK_LIBRARIES "${CMAKE_INSTALL_PREFIX}/${FLANN_LIB_INSTALL_DIR}/libflann_cpp.so"
        INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include"
    )
    # Create alias so PCL can find it as flann_cpp
    add_library(flann::flann_cpp ALIAS flann_cpp_export)
    install(TARGETS flann_cpp_export EXPORT ${targets_export_name})
else()
    install (
        TARGETS ${flann_install_targets}
        EXPORT ${targets_export_name}
        INCLUDES DESTINATION include
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
endif()


install (
    DIRECTORY flann
    DESTINATION include
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)
