cmake_minimum_required(VERSION 3.22.1)
project(fastdds-prefab VERSION 2.13.4.2)

# Config
set(MIN_SDK_VERSION 24)
set(MAVEN_GROUP_ID io.github.eyr1n)
string(REPLACE "." "/" MAVEN_GROUP_ID_SLASH ${MAVEN_GROUP_ID})
string(REPLACE "-" "_" PROJECT_NAME_UNDERSCORE ${PROJECT_NAME})

set(PREFAB_MODULES_DIR ${CMAKE_CURRENT_BINARY_DIR}/prefab/modules)
set(MAVEN_BUNDLE_DIR ${CMAKE_CURRENT_BINARY_DIR}/maven/${MAVEN_GROUP_ID_SLASH}/${PROJECT_NAME}/${PROJECT_VERSION})

include(ExternalProject)

# Build external projects with specific ABI
function(build_with_abi android_abi)
  set(ext_install_dir ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/install)
  set(android_cmake_args
    -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
    -DCMAKE_BUILD_TYPE=Release
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
    GIT_TAG         v2.2.1
    PREFIX          ${android_abi}
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${ext_install_dir}/Fast-CDR
      -DBUILD_SHARED_LIBS=ON
      ${android_cmake_args}
  )
  install(
    DIRECTORY   ${ext_install_dir}/Fast-CDR/include
    DESTINATION ${PREFAB_MODULES_DIR}/fastcdr/libs/android.${android_abi}
  )
  install(
    FILES
      ${ext_install_dir}/Fast-CDR/lib/libfastcdr.so
      ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/abi.json
    DESTINATION ${PREFAB_MODULES_DIR}/fastcdr/libs/android.${android_abi}
  )

  # Fast-DDS
  ExternalProject_Add(Fast-DDS_${android_abi}
    GIT_REPOSITORY  https://github.com/eProsima/Fast-DDS.git
    GIT_TAG         v2.13.4
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
    DESTINATION ${PREFAB_MODULES_DIR}/fastrtps/libs/android.${android_abi}
  )
  install(
    FILES
      ${ext_install_dir}/Fast-DDS/lib/libfastrtps.so
      ${CMAKE_CURRENT_BINARY_DIR}/${android_abi}/abi.json
    DESTINATION ${PREFAB_MODULES_DIR}/fastrtps/libs/android.${android_abi}
  )
endfunction()

configure_file(${PROJECT_NAME}.pom.in ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.pom @ONLY)
configure_file(AndroidManifest.xml.in AndroidManifest.xml @ONLY)
configure_file(prefab.json.in prefab/prefab.json @ONLY)

build_with_abi(arm64-v8a)
build_with_abi(x86_64)

# Create AAR
install(CODE
  "execute_process(
    COMMAND
      ${CMAKE_COMMAND} -E tar cvf ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.aar --format=zip
      AndroidManifest.xml prefab
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  )"
)

# Sign
install(CODE
  "execute_process(
    COMMAND gpg --batch --yes -ab ${PROJECT_NAME}-${PROJECT_VERSION}.pom
    COMMAND gpg --batch --yes -ab ${PROJECT_NAME}-${PROJECT_VERSION}.aar
    WORKING_DIRECTORY ${MAVEN_BUNDLE_DIR}
  )"
)

# Checksum
install(CODE
  "file(MD5 ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.pom POM_MD5)
  file(WRITE ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.pom.md5 \${POM_MD5})
  file(SHA1 ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.pom POM_SHA1)
  file(WRITE ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.pom.sha1 \${POM_SHA1})

  file(MD5 ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.aar AAR_MD5)
  file(WRITE ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.aar.md5 \${AAR_MD5})
  file(SHA1 ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.aar AAR_SHA1)
  file(WRITE ${MAVEN_BUNDLE_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.aar.sha1 \${AAR_SHA1})"
)

# Create Maven bundle
install(CODE
  "execute_process(
    COMMAND
      ${CMAKE_COMMAND} -E tar cvf ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-${PROJECT_VERSION}.zip --format=zip .
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/maven
  )"
)
