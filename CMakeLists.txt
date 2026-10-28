cmake_minimum_required(VERSION 3.20)

# Ab CMake 4 fehlt sonst -isysroot in compile_commands.json und clang-tidy findet stdio.h nicht.
if(APPLE AND NOT CMAKE_OSX_SYSROOT)
    set(CMAKE_OSX_SYSROOT macosx)
endif()

project(Programmierpraktikum C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
add_compile_options(-Wall -Wextra -pedantic)

# week_N/foo.c wird ein eigenes Programm; Projekte aus mehreren Dateien kommen in einen
# Unterordner week_N/foo/, dessen .c-Dateien zusammen ein Programm ergeben.
file(GLOB weeks LIST_DIRECTORIES true "${CMAKE_SOURCE_DIR}/week_*")
foreach(week_dir IN LISTS weeks)
    if(NOT IS_DIRECTORY "${week_dir}")
        continue()
    endif()
    get_filename_component(week "${week_dir}" NAME)

    file(GLOB single_files CONFIGURE_DEPENDS "${week_dir}/*.c")
    foreach(src IN LISTS single_files)
        get_filename_component(name "${src}" NAME_WE)
        string(MAKE_C_IDENTIFIER "${week}_${name}" target)
        add_executable(${target} "${src}")
        set_target_properties(${target} PROPERTIES OUTPUT_NAME "${name}" RUNTIME_OUTPUT_DIRECTORY "${week_dir}")
    endforeach()

    file(GLOB entries LIST_DIRECTORIES true "${week_dir}/*")
    foreach(sub IN LISTS entries)
        if(IS_DIRECTORY "${sub}")
            file(GLOB_RECURSE sources CONFIGURE_DEPENDS "${sub}/*.c")
            if(sources)
                get_filename_component(name "${sub}" NAME)
                string(MAKE_C_IDENTIFIER "${week}_${name}" target)
                add_executable(${target} ${sources})
                # Nicht in week_dir, dort heißt schon der Ordner selbst so.
                set_target_properties(${target} PROPERTIES OUTPUT_NAME "${name}" RUNTIME_OUTPUT_DIRECTORY "${sub}")
            endif()
        endif()
    endforeach()
endforeach()

file(GLOB_RECURSE c_sources CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/week_*/*.c")
file(GLOB_RECURSE c_headers CONFIGURE_DEPENDS "${CMAKE_SOURCE_DIR}/week_*/*.h")

# Homebrew installiert LLVM keg-only, also nicht im PATH.
find_program(CLANG_TIDY clang-tidy HINTS /opt/homebrew/opt/llvm/bin)
if(CLANG_TIDY)
    add_custom_target(tidy
        COMMAND "${CLANG_TIDY}" -p "${CMAKE_BINARY_DIR}" --quiet ${c_sources}
        WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
        VERBATIM)
endif()

find_program(CLANG_FORMAT clang-format HINTS /opt/homebrew/opt/llvm/bin)
if(CLANG_FORMAT)
    add_custom_target(format
        COMMAND "${CLANG_FORMAT}" -i ${c_sources} ${c_headers}
        VERBATIM)
endif()
