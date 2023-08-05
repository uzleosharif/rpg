cmake_minimum_required(VERSION 3.26)

project(rpg CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(ENABLE_IPO "Enable Interprocedural Optimization, aka Link Time Optimization (LTO)" OFF)
if(ENABLE_IPO)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT result OUTPUT output)
  if(result)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
  else()
    message(SEND_ERROR "IPO is not supported: ${output}")
  endif()
endif()

set(CMAKE_CXX_STANDARD_REQUIRED 20)

set(CMAKE_CXX_FLAGS
    "${CMAKE_CXX_FLAGS} -Werror -stdlib=libc++ \
    -Wall -Wextra -Wshadow -Wnon-virtual-dtor -Wold-style-cast -Wcast-align -Wunused -Woverloaded-virtual -Wpedantic \
    -Wconversion -Wsign-conversion -Wnull-dereference -Wdouble-promotion -Wformat=2 -Wmisleading-indentation")

option(ENABLE_COVERAGE "Enable coverage reporting for gcc/clang" OFF)
if(ENABLE_COVERAGE)
  target_compile_options(${PROJECT_NAME} INTERFACE --coverage -O0 -g)
  target_link_libraries(${PROJECT_NAME} INTERFACE --coverage)
endif()

set(SANITIZERS "")

option(ENABLE_SANITIZER_ADDRESS "Enable address sanitizer" ON)
if(ENABLE_SANITIZER_ADDRESS)
  list(APPEND SANITIZERS "address")
endif()

option(ENABLE_SANITIZER_MEMORY "Enable memory sanitizer" OFF)
if(ENABLE_SANITIZER_MEMORY)
  list(APPEND SANITIZERS "memory")
endif()

option(ENABLE_SANITIZER_THREAD "Enable thread sanitizer" OFF)
if(ENABLE_SANITIZER_THREAD)
  list(APPEND SANITIZERS "thread")
endif()

list(
  JOIN
  SANITIZERS
  ","
  LIST_OF_SANITIZERS)

option(ENABLE_DOXYGEN "Enable doxygen doc builds of source" ON)
if(ENABLE_DOXYGEN)
  set(DOXYGEN_CALLER_GRAPH YES)
  set(DOXYGEN_CALL_GRAPH YES)
  set(DOXYGEN_EXTRACT_ALL YES)

  find_package(Doxygen REQUIRED)
  doxygen_add_docs(docs ${PROJECT_SOURCE_DIR} ALL COMMENT "Generate documentation.")
endif()

option(ENABLE_TESTING "Enable Test Builds" ON)
option(ENABLE_FUZZING "Enable Fuzzing Builds" ON)

find_package(spdlog REQUIRED)
find_package(imgui REQUIRED)
find_package(SFML REQUIRED)

add_executable(${PROJECT_NAME} src/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE spdlog::spdlog imgui::imgui sfml::sfml)
target_include_directories(${PROJECT_NAME} PRIVATE include/)

if(LIST_OF_SANITIZERS)
  if(NOT
     "${LIST_OF_SANITIZERS}"
     STREQUAL
     "")
    target_compile_options(${PROJECT_NAME} PRIVATE -fsanitize=${LIST_OF_SANITIZERS})
    target_link_libraries(${PROJECT_NAME} PRIVATE -fsanitize=${LIST_OF_SANITIZERS})
  endif()
endif()

if(ENABLE_TESTING)
  enable_testing()
  message("Building Tests.")
  add_subdirectory(test)
endif()

if(ENABLE_FUZZING)
  message("Building Fuzz Tests.")
  add_subdirectory(fuzz_test)
endif()
