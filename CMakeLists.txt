project(quickjs LANGUAGES C)

file(STRINGS VERSION version)

set(quickjs_src quickjs.c libbf.c libunicode.c libregexp.c cutils.c quickjs-libc.c)
set(quickjs_def CONFIG_VERSION="${version}" _GNU_SOURCE CONFIG_BIGNUM)

if (CMAKE_BUILD_TYPE EQUAL "DEBUG")
    set(_QUICKJS_DUMPLEAKS_DEFAULT ON)
else()
    set(_QUICKJS_DUMPLEAKS_DEFAULT OFF)
endif()
option(QUICKJS_DUMPLEAKS "Enable the 'dumpleaks' option in QuickJS" ${_QUICKJS_DUMPLEAKS_DEFAULT})
if (QUICKJS_DUMPLEAKS)
    set(quickjs_def ${quickjs_def} DUMP_LEAKS=1)
endif()
option(QUICKJS_DISABLE_STACK_CHECK "Disable stack checking, e.g. for use with threads / fibers" ON)
if (QUICKJS_DISABLE_STACK_CHECK)
    set(quickjs_def ${quickjs_def} -DDISABLE_STACK_CHECK=1)
endif()

add_library(quickjs ${quickjs_src})
target_compile_definitions(quickjs PRIVATE ${quickjs_def})
set_target_properties(quickjs PROPERTIES PUBLIC_HEADER "quickjs.h;quickjs-libc.h")
target_include_directories(quickjs PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})


# Some newer MINGW platforms aren't automatically detected 
if (MSYS OR MINGW OR MINGW64 OR "${CMAKE_HOST_SYSTEM_NAME}" MATCHES "MINGW")
  add_library(quickjs_obj OBJECT ${quickjs_src})
  target_compile_definitions(quickjs_obj PRIVATE ${quickjs_def} )
  set_property(TARGET quickjs_obj PROPERTY POSITION_INDEPENDENT_CODE ON)

  add_library(quickjs_s SHARED "quickjs.h")
  set_target_properties(quickjs_s PROPERTIES LINKER_LANGUAGE C)
  target_link_libraries(quickjs_s
    PRIVATE 
    -static -static-libgcc
    -Wl,--out-implib,libquickjs_s.lib
    -Wl,--whole-archive
    quickjs_obj
    -Wl,--no-whole-archive
  )

  set_target_properties(quickjs_s PROPERTIES OUTPUT_NAME "quickjs_s")
  set_target_properties(quickjs_s PROPERTIES SUFFIX ".dll")
else()

endif()


if(UNIX OR MINGW)
  find_package(Threads)
  target_link_libraries(quickjs ${CMAKE_DL_LIBS} m Threads::Threads)
endif()

# install
include(GNUInstallDirs)
install(TARGETS quickjs
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/quickjs
        ARCHIVE       DESTINATION ${CMAKE_INSTALL_LIBDIR}/quickjs
        LIBRARY       DESTINATION ${CMAKE_INSTALL_LIBDIR}/quickjs
        )
