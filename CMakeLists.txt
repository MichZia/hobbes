cmake_minimum_required(VERSION 3.4.1)
project(hobbes)
set (CMAKE_CXX_STANDARD 11)

find_package(ZLIB REQUIRED)
include_directories(${ZLIB_INCLUDE_DIRS})

find_package(LLVM REQUIRED CONFIG)
include_directories(${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

if (${LLVM_PACKAGE_VERSION} VERSION_LESS "3.6")
  set(jit_lib jit)
else()
  set(jit_lib mcjit)
endif()
# 'x86' module should have covered 'x86asmparser' and 'x86codegen' already, but using 'x86' alone didn't work
llvm_map_components_to_libnames(llvm_libs X86asmparser X86codegen X86 ipo ${jit_lib})

file(GLOB_RECURSE lib_headers lib/*.H)
file(GLOB_RECURSE lib_source lib/*.C)
set(lib_files ${lib_headers} ${lib_source})
include_directories(include)

add_library(hobbes STATIC ${lib_files})
target_link_libraries(hobbes ${llvm_libs} ${ZLIB_LIBRARIES})
add_library(hobbes-pic STATIC ${lib_files})
target_link_libraries(hobbes-pic ${llvm_libs} ${ZLIB_LIBRARIES})
set_property(TARGET hobbes-pic PROPERTY POSITION_INDEPENDENT_CODE TRUE)

file(GLOB test_files test/*.C)
file(GLOB hi_files bin/hi/*.C)
file(GLOB_RECURSE hog_files bin/hog/*.C)

if(UNIX AND NOT APPLE)
  set(LINUX TRUE)
endif()

if(APPLE)
  set(sys_libs pthread dl)
  set(cxx_flags "-Wreorder")
  include_directories(/usr/local/opt/readline/include)
  link_directories(/usr/local/opt/readline/lib)
endif()
if(LINUX)
  set(sys_libs pthread dl rt)
  set(cxx_flags "-Werror=old-style-cast -Werror -Wall -Wextra -Winit-self -Wreturn-type -Wunused-variable -Wsign-compare -Warray-bounds -Wunknown-pragmas -Wuninitialized -Wstrict-aliasing -Wunused-value -Wunused-label -Wswitch -Wcast-align -Wctor-dtor-privacy -Wmissing-noreturn -Wunused-parameter -Wreorder")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${cxx_flags}")
set(CMAKE_CXX_FLAGS_DEBUG "-g")
set(CMAKE_CXX_FLAGS_RELEASE "-O2")

add_executable(hi ${hi_files})
target_link_libraries(hi hobbes ncurses readline ${sys_libs})
add_executable(hog ${hog_files})
target_link_libraries(hog hobbes ncurses ${sys_libs})

enable_testing()
add_executable(hobbes-test ${test_files})
target_link_libraries(hobbes-test hobbes ncurses ${sys_libs})
add_test(hobbes-test hobbes-test)
include(FindPythonInterp)
set_property(TARGET hobbes-test PROPERTY COMPILE_FLAGS "-DPYTHON_EXECUTABLE=\"${PYTHON_EXECUTABLE}\" -DSCRIPT_DIR=\"${CMAKE_SOURCE_DIR}/scripts/\"")

install(TARGETS hobbes hobbes-pic DESTINATION "lib")
install(TARGETS hi hog DESTINATION "bin")
install(DIRECTORY "include/hobbes" DESTINATION "include")
install(DIRECTORY "scripts" DESTINATION "scripts")

