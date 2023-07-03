cmake_minimum_required(VERSION 3.22.1)
project(fastdds-prefab VERSION 1.0.0)

# Config
set(MIN_SDK_VERSION 24)

set(android_aar_name fastdds-prefab-${CMAKE_PROJECT_VERSION})
set(prefab_modules_dir ${CMAKE_CURRENT_BINARY_DIR}/${android_aar_name}/prefab/modules)

include(ExternalProject)

# Build external projects with specific ABI
function(build_with_abi android_abi)
  set(ext_install_dir ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/install)
  set(android_cmake_args
    -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
    -DANDROID_ABI=${android_abi}
    -DANDROID_PLATFORM=android-${MIN_SDK_VERSION}
    -DANDROID_STL=c++_shared
  )

  configure_file(abi.json.in ${android_abi}/abi.json @ONLY)

  # foonathan_memory_vendor
  ExternalProject_Add(foonathan_memory_vendor_${android_abi}
    GIT_REPOSITORY  https://github.com/eProsima/foonathan_memory_vendor.git
    GIT_TAG         v1.3.1
    PREFIX          ${android_abi}
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${ext_install_dir}/foonathan_memory_vendor
      -DBUILD_SHARED_LIBS=OFF
      ${android_cmake_args}
  )

  # Fast-CDR
  ExternalProject_Add(Fast-CDR_${android_abi}
    GIT_REPOSITORY  https://github.com/eProsima/Fast-CDR.git
    GIT_TAG         v1.1.0
    PREFIX          ${android_abi}
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${ext_install_dir}/Fast-CDR
      -DBUILD_SHARED_LIBS=ON
      ${android_cmake_args}
  )
  install(
    DIRECTORY   ${ext_install_dir}/Fast-CDR/include
    DESTINATION ${prefab_modules_dir}/fastcdr/libs/android.${android_abi}
  )
  install(
    FILES
      ${ext_install_dir}/Fast-CDR/lib/libfastcdr.so
      ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/abi.json
    DESTINATION ${prefab_modules_dir}/fastcdr/libs/android.${android_abi}
  )

  # Fast-DDS
  ExternalProject_Add(Fast-DDS_${android_abi}
    GIT_REPOSITORY  https://github.com/eProsima/Fast-DDS.git
    GIT_TAG         v2.10.1
    PREFIX          ${android_abi}
    DEPENDS         foonathan_memory_vendor_${android_abi} Fast-CDR_${android_abi}
    LIST_SEPARATOR |
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${ext_install_dir}/Fast-DDS
      -DCMAKE_FIND_ROOT_PATH=${ext_install_dir}/foonathan_memory_vendor|${ext_install_dir}/Fast-CDR
      -DBUILD_SHARED_LIBS=ON
      -DTHIRDPARTY_fastcdr=OFF
      -DTHIRDPARTY_Asio=FORCE
      -DTHIRDPARTY_TinyXML2=FORCE
      -DTHIRDPARTY_android-ifaddrs=FORCE
      -DSHM_TRANSPORT_DEFAULT=OFF
      ${android_cmake_args}
  )
  install(
    DIRECTORY   ${ext_install_dir}/Fast-DDS/include
    DESTINATION ${prefab_modules_dir}/fastrtps/libs/android.${android_abi}
  )
  install(
    FILES
      ${ext_install_dir}/Fast-DDS/lib/libfastrtps.so
      ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/abi.json
    DESTINATION ${prefab_modules_dir}/fastrtps/libs/android.${android_abi}
  )
endfunction()

configure_file(fastdds-prefab.pom.in ${android_aar_name}.pom @ONLY)
configure_file(prefab.json.in ${android_aar_name}/prefab/prefab.json @ONLY)
install(FILES AndroidManifest.xml DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/${android_aar_name})

build_with_abi(arm64-v8a)
build_with_abi(x86_64)

# Create AAR
install(CODE
  "execute_process(
    COMMAND
      ${CMAKE_COMMAND} -E tar cvf ${CMAKE_CURRENT_BINARY_DIR}/${android_aar_name}.aar --format=zip
      prefab AndroidManifest.xml
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${android_aar_name}
  )"
)

# Create Maven package
install(CODE
  "execute_process(
    COMMAND
      ${CMAKE_COMMAND} -E tar cvf ${CMAKE_CURRENT_BINARY_DIR}/${android_aar_name}.zip --format=zip
      ${android_aar_name}.aar ${android_aar_name}.pom
  )"
)
