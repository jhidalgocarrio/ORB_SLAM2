cmake_minimum_required(VERSION 2.8)
project(ORB_SLAM2)

IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE Release)
ENDIF()

MESSAGE("Build type: " ${CMAKE_BUILD_TYPE})

# Check C++11 or C++0x support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
   add_definitions(-DCOMPILEDWITHC11)
   message(STATUS "Using flag -std=c++11.")
elseif(COMPILER_SUPPORTS_CXX0X)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
   add_definitions(-DCOMPILEDWITHC0X)
   message(STATUS "Using flag -std=c++0x.")
else()
   message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules)

find_package(OpenCV 2.4.3 REQUIRED)
find_package(Eigen3 3.1.0 REQUIRED)
find_package(Pangolin REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(G2O REQUIRED g2o)

if(G2O_FOUND)
    message(STATUS "g2o include directory." ${G2O_INCLUDE_DIRS})
endif()

include_directories(
${PROJECT_SOURCE_DIR}
${PROJECT_SOURCE_DIR}/include
${EIGEN3_INCLUDE_DIR}
${Pangolin_INCLUDE_DIRS}
${G2O_INCLUDE_DIRS}
)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)

add_library(${PROJECT_NAME} SHARED
src/System.cc
src/Tracking.cc
src/LocalMapping.cc
src/LoopClosing.cc
src/ORBextractor.cc
src/ORBmatcher.cc
src/FrameDrawer.cc
src/Converter.cc
src/MapPoint.cc
src/KeyFrame.cc
src/Map.cc
src/MapDrawer.cc
src/Optimizer.cc
src/PnPsolver.cc
src/Frame.cc
src/KeyFrameDatabase.cc
src/Sim3Solver.cc
src/Initializer.cc
src/Viewer.cc
)

target_link_libraries(${PROJECT_NAME}
${OpenCV_LIBS}
${EIGEN3_LIBS}
${Pangolin_LIBRARIES}
${PROJECT_SOURCE_DIR}/Thirdparty/DBoW2/lib/libDBoW2.so
${G2O_LIBRARIES}
${G2O_LDFLAGS}
)

# Build examples

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/RGB-D)

add_executable(rgbd_tum
Examples/RGB-D/rgbd_tum.cc)
target_link_libraries(rgbd_tum ${PROJECT_NAME})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/Stereo)

add_executable(stereo_kitti
Examples/Stereo/stereo_kitti.cc)
target_link_libraries(stereo_kitti ${PROJECT_NAME})

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/Examples/Monocular)

add_executable(mono_tum
Examples/Monocular/mono_tum.cc)
target_link_libraries(mono_tum ${PROJECT_NAME})

add_executable(mono_kitti
Examples/Monocular/mono_kitti.cc)
target_link_libraries(mono_kitti ${PROJECT_NAME})

if(NOT WIN32)
    option(ORB_SLAM2_BUILD_PKGCONFIG "Build pkg-config .pc file for ORB_SLAM2" ON)
endif(NOT WIN32)

macro(libraries_for_pkgconfig VARNAME)
    foreach(__lib ${ARGN})
        string(STRIP __lib ${__lib})
        string(SUBSTRING ${__lib} 0 1 __lib_is_absolute)
        if (__lib_is_absolute STREQUAL "/")
            get_filename_component(__lib_path ${__lib} PATH)
            get_filename_component(__lib_name ${__lib} NAME_WE)
            string(REGEX REPLACE "^lib" "" __lib_name "${__lib_name}")
            set(${VARNAME} "${${VARNAME}} -L${__lib_path} -l${__lib_name}")
        else()
            set(${VARNAME} "${${VARNAME}} ${__lib}")
        endif()
    endforeach()
endmacro()

# Generate pkgconfig pc file
if(ORB_SLAM2_BUILD_PKGCONFIG)
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/ORB_SLAM2.pc.in)
        message("-- Found: ${CMAKE_CURRENT_SOURCE_DIR}/ORB_SLAM2.pc.in")

        # Set the target name
        set(TARGET_NAME ORB_SLAM2)

        # Dependency for ORB_SLAM2
        libraries_for_pkgconfig(${TARGET_NAME}_PKGCONFIG_LIBS ${LAPACK_LIBRARIES})

        set(${TARGET_NAME}_PKGCONFIG_CFLAGS "${${TARGET_NAME}_PKGCONFIG_CFLAGS}")

        set(PKGCONFIG_CFLAGS ${${TARGET_NAME}_PKGCONFIG_CFLAGS})
        set(PKGCONFIG_LIBS ${${TARGET_NAME}_PKGCONFIG_LIBS})

        configure_file(ORB_SLAM2.pc.in ORB_SLAM2.pc @ONLY)
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/ORB_SLAM2.pc
            DESTINATION lib/pkgconfig)
    else()
        message("-- pkg-config: ${CMAKE_CURRENT_SOURCE_DIR}/ORB_SLAM2.pc.in is not available for configuration")
    endif()

endif(ORB_SLAM2_BUILD_PKGCONFIG)

