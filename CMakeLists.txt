cmake_minimum_required(VERSION 3.5 FATAL_ERROR)
project(lua)

set(LUA_VERSION "5.3.5" CACHE STRING "Version of Lua to be built.")
set(LUA_PLATFORM "" CACHE STRING "Plafrom of Lua built for.")

if(NOT LUA_PLATFORM)
  string(TOLOWER ${CPACK_SYSTEM_NAME} LUA_PLATFORM)
endif()

#
# Download and unpack lua packages.
#

function(fetch_lua VERSION OUTPUT)
  string(REPLACE "." "_" VERSION_ID "${VERSION}")

  message(STATUS "Downloading Lua ${VERSION}")

  set(HOST "https://www.lua.org/ftp/")

  set(PACKAGE_5_3_5 112eb10ff04d1b4c9898e121d6bdf54a81482447 lua-5.3.5.tar.gz)
  set(PACKAGE_5_3_4 79790cfd40e09ba796b01a571d4d63b52b1cd950 lua-5.3.4.tar.gz)
  set(PACKAGE_5_3_3 a0341bc3d1415b814cc738b2ec01ae56045d64ef lua-5.3.3.tar.gz)
  set(PACKAGE_5_3_2 7a47adef554fdca7d0c5536148de34579134a973 lua-5.3.2.tar.gz)
  set(PACKAGE_5_3_1 1676c6a041d90b6982db8cef1e5fb26000ab6dee lua-5.3.1.tar.gz)
  set(PACKAGE_5_3_0 1c46d1c78c44039939e820126b86a6ae12dadfba lua-5.3.0.tar.gz)
  set(PACKAGE_5_2_4 ef15259421197e3d85b7d6e4871b8c26fd82c1cf lua-5.2.4.tar.gz)
  set(PACKAGE_5_2_3 926b7907bc8d274e063d42804666b40a3f3c124c lua-5.2.3.tar.gz)
  set(PACKAGE_5_2_2 0857e41e5579726a4cb96732e80d7aa47165eaf5 lua-5.2.2.tar.gz)
  set(PACKAGE_5_2_1 6bb1b0a39b6a5484b71a83323c690154f86b2021 lua-5.2.1.tar.gz)
  set(PACKAGE_5_2_0 08f84c355cdd646f617f09cebea48bd832415829 lua-5.2.0.tar.gz)
  set(PACKAGE_5_1_5 b3882111ad02ecc6b972f8c1241647905cb2e3fc lua-5.1.5.tar.gz)
  set(PACKAGE_5_1_4 2b11c8e60306efb7f0734b747588f57995493db7 lua-5.1.4.tar.gz)
  set(PACKAGE_5_1_3 89bc9f5a351402565b8077e8123327e7cd15f004 lua-5.1.3.tar.gz)
  set(PACKAGE_5_1_2 8a460d2d7e70e93cb72bf3d584405464763cb5f0 lua-5.1.2.tar.gz)
  set(PACKAGE_5_1_1 be13878ceef8e1ee7a4201261f0adf09f89f1005 lua-5.1.1.tar.gz)
  set(PACKAGE_5_1_0 1ae9ec317511d525c7999c842ca0b1ddde84e374 lua-5.1.tar.gz)

  set(PACKAGE "${PACKAGE_${VERSION_ID}}")

  list(GET PACKAGE 0 HASH)
  list(GET PACKAGE 1 FILENAME)

  set(TO "${CMAKE_SOURCE_DIR}/tarballs/${FILENAME}")
  
  file(DOWNLOAD "${HOST}${FILENAME}" ${TO} SHOW_PROGRESS EXPECTED_HASH SHA1=${HASH} TLS_VERIFY on)
  set(${OUTPUT} ${TO} PARENT_SCOPE)
endfunction(fetch_lua)


fetch_lua(${LUA_VERSION} TARBALL)

execute_process(
  COMMAND ${CMAKE_COMMAND} -E tar xzf ${TARBALL}
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

#
# Setup lua projects.
#

function(version_to_abi VERSION OUTPUT)
  string(SUBSTRING ${VERSION} 0 3 ABI)
  string(REPLACE "." "" ABI "${ABI}")
  set(${OUTPUT} "${ABI}" PARENT_SCOPE)
endfunction(version_to_abi)

# add_lua(5.1.5 path/to/lua)
function(add_lua VERSION ROOT OUTPUT)
  version_to_abi(${VERSION} ABI)

  message("called add_lua(${VERSION} ${ROOT})")
  
  include(cmake/lua-${ABI}.cmake) # uses ROOT
  if (MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
  endif()

  # Shared library
  set(LIBNAME lua${ABI})
  add_library(${LIBNAME} SHARED ${LIBRARY_FILES})
  if (MSVC)
    target_compile_definitions(${LIBNAME} PRIVATE -D_CRT_SECURE_NO_WARNINGS -DLUA_BUILD_AS_DLL)
    target_compile_options(${LIBNAME} PRIVATE /wd4334)
  else()
    target_compile_options(${LIBNAME} PRIVATE -Wno-deprecated-declarations -Wno-empty-body)
  endif()

  # Lua executable
  add_executable(lua ${ROOT}/src/lua.c)
  set_target_properties(lua PROPERTIES OUTPUT_NAME lua)
  target_link_libraries(lua ${LIBNAME})

  # Luac executable
  add_executable(luac ${LUAC_FILES})
  target_link_libraries(luac ${LIBNAME})

  # Install files
  set(PREFIX "${CMAKE_SOURCE_DIR}/install/${VERSION}")
  install(
    TARGETS lua luac ${LIBNAME}
    RUNTIME DESTINATION "bin"
    LIBRARY DESTINATION "bin"
    ARCHIVE DESTINATION "lib"
  )
  install(FILES ${LUA_HEADERS} DESTINATION "include")
  install(DIRECTORY ${ROOT}/doc DESTINATION "doc")
  install(FILES ${ROOT}/README DESTINATION "doc")
  set(${OUTPUT} "${PREFIX}" PARENT_SCOPE)
endfunction(add_lua)


add_lua(${LUA_VERSION} ${CMAKE_BINARY_DIR}/lua-${LUA_VERSION} INSTALL_PREFIX)

#
# Packaging
#

set(CPACK_SOURCE_GENERATOR "ZIP;TGZ")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Lua ${LUA_VERSION} binary.")
set(CPACK_PACKAGE_VERSION ${LUA_VERSION})
set(CPACK_SYSTEM_NAME "${LUA_PLATFORM}")

include(CPack)
