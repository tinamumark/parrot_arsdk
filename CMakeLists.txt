cmake_minimum_required(VERSION 3.12)
project(parrot_arsdk)

###########
## Build ##
###########

# use the build space for downloading/building the package
# we populate the results to devel/install spaces after the build
set(ARDRONESDK3_DEVEL_PATH ${CMAKE_CURRENT_BINARY_DIR}/arsdk)
set(ARDRONESDK3_PATH ${ARDRONESDK3_DEVEL_PATH}/src/ARSDKBuildUtils/out/arsdk-native/staging/usr)

# 3.14.0 patch 1
set(ARSDK_ARCHIVE arsdk_3_14_0_p1_stripped.tgz)
set(ARSDK_ARCHIVE_MD5 f73099c420dab7c1ad29a01b1df3ca3c)

# Determine the architecure of the host in a robust way
execute_process(COMMAND python ${PROJECT_SOURCE_DIR}/script/get_arch.py OUTPUT_VARIABLE BUILD_HOST_ARCH)
message(STATUS "Host architecure: ${BUILD_HOST_ARCH}")

add_custom_target(ARSDK_MKDIR
  COMMAND ${CMAKE_COMMAND} -E make_directory ${ARDRONESDK3_DEVEL_PATH})

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

include(ExternalProject)
ExternalProject_Add(ARSDKBuildUtils
    DEPENDS ARSDK_MKDIR
    URL ${PROJECT_SOURCE_DIR}/sdk/${ARSDK_ARCHIVE}
    URL_MD5 ${ARSDK_ARCHIVE_MD5}
    PREFIX ${ARDRONESDK3_DEVEL_PATH}
    CONFIGURE_COMMAND echo "No configure"
    BUILD_COMMAND TARGET_ARCH=${BUILD_HOST_ARCH} ./build.sh -p arsdk-native -t build-sdk -j --no-color
    INSTALL_COMMAND echo "No SDK install command"
    BUILD_IN_SOURCE 1
)

add_custom_target(ARSDKPopulate ALL
  COMMENT "Populating ARDroneSDK to devel space: ${ARDRONESDK3_DEVEL_PATH}"
  WORKING_DIRECTORY ${ARDRONESDK3_DEVEL_PATH}/src/ARSDKBuildUtils
  # Populate the build results into the devel space
  COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/include
  COMMAND cp -rp ./out/arsdk-native/staging/usr/include/* ${PROJECT_BINARY_DIR}/include
  COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/lib
  COMMAND cp -rp ./out/arsdk-native/staging/usr/lib/* ${PROJECT_BINARY_DIR}/lib
  COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/share
  COMMAND cp -rp ./out/arsdk-native/staging/usr/share/* ${PROJECT_BINARY_DIR}/share
  #COMMAND ${CMAKE_COMMAND} -E make_directory ${PROJECT_BINARY_DIR}/bin
  #COMMAND cp -rp ./out/Unix-base/staging/usr/bin/* ${PROJECT_BINARY_DIR}/bin
)
add_dependencies(ARSDKPopulate ARSDKBuildUtils)

set(BEBOP_LIBS
  libarcommands.so
  libarcontroller.so
  libardatatransfer.so
  libardiscovery.so
  libarmavlink.so
  libarmedia.so
  libarnetwork.so
  libarnetworkal.so
  libarsal.so
  libarstream.so
  libarstream2.so
  libarupdater.so
  libarutils.so
  libcrypto.so
  libcurl.so
  libfutils.so
  libjson-c.so
  libjson.so
  libmux.so
  libpomp.so
  libpuf.so
  librtsp.so
  libsdp.so
  libssl.so
  libtar.so
  libtls.so
  libulog.so
)
set(BEBOP_LIBS_FULL_PATH ${BEBOP_LIBS})
list(TRANSFORM BEBOP_LIBS_FULL_PATH PREPEND ${PROJECT_BINARY_DIR}/lib/)

add_library(arsdk INTERFACE)
add_dependencies(arsdk ARSDKPopulate)


write_basic_package_version_file(
    arsdkConfigVersion.cmake
    VERSION 1.0.0
    COMPATIBILITY SameMajorVersion
)

install(
  TARGETS arsdk 
  EXPORT arsdkTargets
  DESTINATION lib/
)

configure_package_config_file(
    cmake/arsdk.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/arsdkConfig.cmake
    INSTALL_DESTINATION lib/cmake/arsdk
)

install(EXPORT arsdkTargets
    FILE arsdkTargets.cmake
    NAMESPACE arsdk::
    DESTINATION lib/cmake/arsdk
)

install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/arsdkConfig.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/arsdkConfigVersion.cmake
    DESTINATION lib/cmake/arsdk
)

install(
    FILES ${BEBOP_LIBS_FULL_PATH} DESTINATION lib
)
install(
  DIRECTORY ${PROJECT_BINARY_DIR}/include/ DESTINATION include 
)