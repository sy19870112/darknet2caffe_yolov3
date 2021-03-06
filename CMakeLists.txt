cmake_minimum_required(VERSION 2.8)
project(sysDetectSpeed)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")	# -std=gnu++11

# setup CUDA
find_package(CUDA)

set(
	CUDA_NVCC_FLAGS
	${CUDA_NVCC_FLAGS}; 
    -O3 
	-gencode arch=compute_53,code=sm_53  #tegra tx1 
        -gencode arch=compute_61,code=sm_61  #gtx 1060
	-gencode arch=compute_62,code=sm_62  #tegra tx2
        -gencode arch=compute_75,code=[sm_75,compute_75]
        
)

# setup project output paths
set(PROJECT_OUTPUT_DIR  ${PROJECT_BINARY_DIR}/${CMAKE_SYSTEM_PROCESSOR})
set(PROJECT_INCLUDE_DIR ${PROJECT_OUTPUT_DIR}/include)

file(MAKE_DIRECTORY ${PROJECT_INCLUDE_DIR})
file(MAKE_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)

message("-- system arch:  ${CMAKE_SYSTEM_PROCESSOR}")
message("-- output path:  ${PROJECT_OUTPUT_DIR}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_OUTPUT_DIR}/lib)


# build C/C++ interface
include_directories(${PROJECT_INCLUDE_DIR} ${GIE_PATH}/include)
include_directories(${PROJECT_INCLUDE_DIR} 
	/home/hs/software/caffe/include 
        /home/hs/software/caffe/.build_release/src
        /usr/include/openblas/
)


file(GLOB inferenceSources *.cpp *.cu )
file(GLOB inferenceIncludes *.h )

cuda_add_library(sysDetectSpeed SHARED ${inferenceSources})
target_link_libraries(sysDetectSpeed 
        /home/hs/software/caffe/.build_release/lib/libcaffe.so
        /usr/lib64/libglog.so
        /usr/lib64/libgflags.so
        /usr/lib64/libgflags.so.2.1
        /usr/lib64/boost/lib/libboost_system.so
        /root/anaconda3/envs/venv/lib/libstdc++.so.6
        /usr/lib64/libtiff.so
)


# transfer all headers to the include directory
foreach(include ${inferenceIncludes})
	message("-- Copying ${include}")
	configure_file(${include} ${PROJECT_INCLUDE_DIR} COPYONLY)
endforeach()


# create symbolic link for network data
execute_process( COMMAND "${CMAKE_COMMAND}" "-E" "create_symlink" "${PROJECT_SOURCE_DIR}/data/networks" "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/networks" )
  
add_subdirectory(detectnet)


#setup opencv
FIND_PACKAGE(OpenCV REQUIRED)

TARGET_LINK_LIBRARIES(sysDetectSpeed ${OpenCV_LIBS})

# install
foreach(include ${inferenceIncludes})
    install(FILES "${include}" DESTINATION include/sysDetectSpeed)
endforeach()

# install the shared library
install(TARGETS sysDetectSpeed DESTINATION lib/sysDetectSpeed EXPORT sysDetectSpeedConfig)

# install the cmake project, for importing
install(EXPORT sysDetectSpeedConfig DESTINATION share/sysDetectSpeed/cmake)
