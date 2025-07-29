
add_custom_target(tests)
add_custom_target(flann_gtests)

add_dependencies(tests flann_gtests)

set(EXECUTABLE_OUTPUT_PATH ${TEST_OUTPUT_PATH})

# ISAMORE LLVM IR generation for FLANN tests
if(FLANN_ENABLE_LLVM_IR)
  # Function to add LLVM IR generation for a FLANN test
  function(FLANN_ADD_TEST_LLVM_IR name)
    # Skip cuda tests
    if(name MATCHES "cuda")
      return()
    endif()
    
    # Find the test source file
    set(test_src "${CMAKE_CURRENT_SOURCE_DIR}/${name}.cpp")
    if(NOT EXISTS ${test_src})
      message(WARNING "Cannot find source file for test ${name}: ${test_src}")
      return()
    endif()
    
    # Generate LLVM IR for the test
    add_custom_command(
      OUTPUT ${CMAKE_BINARY_DIR}/test/llvm/${name}.ll
      COMMAND ${CMAKE_BINARY_DIR}/../compile_flann_to_ll.sh 
              ${test_src}
              ${CMAKE_BINARY_DIR}/test/llvm/${name}.ll
              "-I${CMAKE_SOURCE_DIR}/src/cpp -I${CMAKE_BINARY_DIR}/src/cpp -I${HDF5_INCLUDE_DIR}"
              "-DFLANN_EXPORT"
      DEPENDS ${test_src}
      COMMENT "Generating LLVM IR: ${name}.ll"
    )
    
    # Instrument the test
    add_custom_command(
      OUTPUT ${CMAKE_BINARY_DIR}/test/llvm/${name}_instrumented.ll
      COMMAND ${OPT} -load-pass-plugin="${PASS_PATH}" -passes=bb_instrument 
              ${CMAKE_BINARY_DIR}/test/llvm/${name}.ll 
              -o ${CMAKE_BINARY_DIR}/test/llvm/${name}_instrumented.ll
      DEPENDS ${CMAKE_BINARY_DIR}/test/llvm/${name}.ll
      COMMENT "Instrumenting: ${name}_instrumented.ll"
    )
    
    # Link with FLANN libraries and build executable
    add_custom_command(
      OUTPUT ${CMAKE_BINARY_DIR}/test/${name}_instrumented
      COMMAND ${LLVM_LINK} ${CMAKE_BINARY_DIR}/test/llvm/${name}_instrumented.ll 
              ${CMAKE_BINARY_DIR}/llvm/flann_instrumented.bc
              -o ${CMAKE_BINARY_DIR}/test/llvm/${name}_linked.bc
      COMMAND ${CLANG} ${CMAKE_BINARY_DIR}/test/llvm/${name}_linked.bc
              -o ${CMAKE_BINARY_DIR}/test/${name}_instrumented
              -lm -lpthread -lgtest -lgtest_main
              -L${CMAKE_BINARY_DIR} -Wl,-rpath,${CMAKE_BINARY_DIR}
              -L$ENV{HOME}/.local/lib -Wl,-rpath,$ENV{HOME}/.local/lib
              -llz4 ${HDF5_LIBRARIES}
      DEPENDS ${CMAKE_BINARY_DIR}/test/llvm/${name}_instrumented.ll ${CMAKE_BINARY_DIR}/llvm/flann_instrumented.bc
      COMMENT "Building instrumented executable: ${name}_instrumented"
    )
    
    add_custom_target(${name}_instrumented_exe ALL
      DEPENDS ${CMAKE_BINARY_DIR}/test/${name}_instrumented
    )
  endfunction()
  
  # Override flann_add_gtest to automatically add LLVM IR generation
  include(${CMAKE_SOURCE_DIR}/cmake/flann_utils.cmake)
  
  macro(flann_add_gtest_original exe src)
    flann_add_gtest(${exe} ${src} ${ARGN})
  endmacro()
  
  # Redefine flann_add_gtest to include LLVM IR generation
  macro(flann_add_gtest exe src)
    # Call original macro
    add_executable(${exe} EXCLUDE_FROM_ALL ${src})
    target_link_libraries(${exe} ${googletest_LIBRARIES} ${ARGN})
    add_dependencies(${exe} googletest)
    add_dependencies(flann_gtests ${exe})
    string(REPLACE "/" "_" _testname ${exe})
    add_test(
        NAME test_${_testname}
        COMMAND ${exe} --gtest_print_time
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/test
    )
    
    # Add LLVM IR generation
    if(FLANN_ENABLE_LLVM_IR)
      FLANN_ADD_TEST_LLVM_IR(${exe})
    endif()
  endmacro()
endif()

if (HDF5_FOUND)
    include_directories(${HDF5_INCLUDE_DIR})

    set(TEST_LIBRARIES "${HDF5_LIBRARIES}")
    if (HDF5_IS_PARALLEL)
        set(TEST_LIBRARIES "${TEST_LIBRARIES};${MPI_LIBRARIES}")
    endif()

    flann_add_gtest(flann_linear_test flann_linear_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_kdtree_test flann_kdtree_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_kmeans_test flann_kmeans_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_kdtree_single_test flann_kdtree_single_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_hierarchical_test flann_hierarchical_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_lsh_test flann_lsh_test.cpp flann_cpp ${TEST_LIBRARIES})
    flann_add_gtest(flann_autotuned_test flann_autotuned_test.cpp flann_cpp ${TEST_LIBRARIES})
    if (OPENMP_FOUND)
        flann_add_gtest(flann_multithreaded_test flann_multithreaded_test.cpp flann_cpp ${TEST_LIBRARIES})
    endif()

endif()

if (HDF5_FOUND AND BUILD_CUDA_LIB)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-Xcompiler;-fPIC;-Xcudafe \"--diag_suppress=partial_override\" ;-gencode=arch=compute_52,code=\"sm_52,compute_52\";-gencode=arch=compute_61,code=\"sm_61,compute_61\"" )
    if (NVCC_COMPILER_BINDIR)
        set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};--compiler-bindir=${NVCC_COMPILER_BINDIR}")
    endif()
    flann_add_cuda_gtest(flann_cuda_test flann_cuda_test.cu flann_cpp ${HDF5_LIBRARIES} flann_cuda)
endif()


#---------- pyunit tests --------------
if (BUILD_PYTHON_BINDINGS) 
    flann_add_pyunit(test_nn.py)
    flann_add_pyunit(test_nn_index.py)
    flann_add_pyunit(test_index_save.py)
    flann_add_pyunit(test_nn_autotune.py)
    flann_add_pyunit(test_clustering.py)
endif()

#---------- ruby spec ----------------
#if (BUILD_C_BINDINGS)
#    add_custom_target(flann_ruby_spec
#                    COMMAND bundle exec rake spec
#                    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/src/ruby)
#    add_dependencies(test flann_ruby_spec)
#endif()
