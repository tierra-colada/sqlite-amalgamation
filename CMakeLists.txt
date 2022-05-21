cmake_minimum_required(VERSION 3.8)
project(SQLite3
    VERSION   3.31.1
    LANGUAGES C
    )

include(GNUInstallDirs)

#------------------------------------------------------------------------------
# build options and optional modules:
option(SQLITE_ENABLE_COLUMN_METADATA  "enables column metadata"                           OFF)
option(SQLITE_ENABLE_DBSTAT_VTAB      "enables dbstat virtual table"                      OFF)
option(SQLITE_ENABLE_FTS3             "enables full text searches version 3"              OFF)
option(SQLITE_ENABLE_FTS4             "enables full text searches version 3 & 4"          OFF)
option(SQLITE_ENABLE_FTS5             "enables full text searches version 5"              OFF)
option(SQLITE_ENABLE_GEOPOLY          "enables Geopoly extention"                         OFF)
option(SQLITE_ENABLE_ICU              "enables international components for unicode"      OFF)
option(SQLITE_ENABLE_MATH_FUNCTIONS   "enables the built-in SQL math functions"           ON)
option(SQLITE_ENABLE_RBU              "enables resumable bulk update extension"           OFF)
option(SQLITE_ENABLE_RTREE            "enables R*TRee index extension"                    OFF)
option(SQLITE_ENABLE_STAT4            "enhances query planner under certain situations"   OFF)
option(SQLITE_OMIT_DECLTYPE           "omit declared type of columns"                     ON)
option(SQLITE_OMIT_JSON               "disables JSON SQL functions"                       OFF)
option(SQLITE_OMIT_DEPRECATED         "disables deprecated functions"                     ON)
option(SQLITE_OMIT_PROGRESS_CALLBACK  "disables progress callback functions"              ON)
option(SQLITE_OMIT_SHARED_CACHE       "disables shared cache (for perfomance) functions"  ON)
option(SQLITE_RECOMMENDED_OPTIONS     "compile by SQLite3 recommended options"            ON)
option(SQLITE_USE_URI                 "enables the default URI filename processing"       OFF)
option(SQLITE_BUILD_SHARED            "build as SHARED library"                           ON)
option(SQLITE_BUILD_STATIC            "build as STATIC library"                           ON)

if(NOT BUILD_SHARED_LIBS AND NOT SQLITE_BUILD_SHARED AND NOT SQLITE_BUILD_STATIC)
    message(FATAL_ERROR "Neither STATIC nor SHARED SQlite3 were chosen!")
endif()

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Release or Debug?" FORCE)
endif()

if(SQLITE_ENABLE_COLUMN_METADATA AND SQLITE_OMIT_DECLTYPE)
    message(FATAL_ERROR "please unset the SQLITE_OMIT_DECLTYPE if you want to\
    compile with SQLITE_ENABLE_COLUMN_METADATA,\
    compiling with both options ON, is not recommended.")
endif()

#------------------------------------------------------------------------------

set(src_files sqlite3.c)
set(include_dirs $<INSTALL_INTERFACE:include>)
set(public_headers sqlite3.h sqlite3ext.h)
set(user_options $<BUILD_INTERFACE:
    $<$<BOOL:${SQLITE_ENABLE_COLUMN_METADATA}>:SQLITE_ENABLE_COLUMN_METADATA>
    $<$<BOOL:${SQLITE_ENABLE_DBSTAT_VTAB}>:SQLITE_ENABLE_DBSTAT_VTAB>
    $<$<BOOL:${SQLITE_ENABLE_FTS3}>:SQLITE_ENABLE_FTS3>
    $<$<BOOL:${SQLITE_ENABLE_FTS4}>:SQLITE_ENABLE_FTS4>
    $<$<BOOL:${SQLITE_ENABLE_FTS5}>:SQLITE_ENABLE_FTS5>
    $<$<BOOL:${SQLITE_ENABLE_GEOPOLY}>:SQLITE_ENABLE_GEOPOLY>
    $<$<BOOL:${SQLITE_ENABLE_ICU}>:SQLITE_ENABLE_ICU>
    $<$<BOOL:${SQLITE_ENABLE_MATH_FUNCTIONS}>:SQLITE_ENABLE_MATH_FUNCTIONS>
    $<$<BOOL:${SQLITE_ENABLE_RBU}>:SQLITE_ENABLE_RBU>
    $<$<BOOL:${SQLITE_ENABLE_RTREE}>:SQLITE_ENABLE_RTREE>
    $<$<BOOL:${SQLITE_ENABLE_STAT4}>:SQLITE_ENABLE_STAT4>
    $<$<BOOL:${SQLITE_OMIT_DECLTYPE}>:SQLITE_OMIT_DECLTYPE>
    $<$<BOOL:${SQLITE_OMIT_JSON}>:SQLITE_OMIT_JSON>
    $<$<BOOL:${SQLITE_OMIT_DEPRECATED}>:SQLITE_OMIT_DEPRECATED>
    $<$<BOOL:${SQLITE_OMIT_PROGRESS_CALLBACK}>:SQLITE_OMIT_PROGRESS_CALLBACK>
    $<$<BOOL:${SQLITE_OMIT_SHARED_CACHE}>:SQLITE_OMIT_SHARED_CACHE>
    $<$<BOOL:${SQLITE_USE_URI}>:SQLITE_USE_URI>
    $<$<BOOL:${SQLITE_RECOMMENDED_OPTIONS}>:
        SQLITE_DEFAULT_MEMSTATUS=0
        SQLITE_DEFAULT_WAL_SYNCHRONOUS=1
        SQLITE_DQS=0
        SQLITE_LIKE_DOESNT_MATCH_BLOBS
        SQLITE_MAX_EXPR_DEPTH=0
        SQLITE_USE_ALLOCA
        >
    >)

# SQLite3 as SHARED library
if(SQLITE_BUILD_SHARED OR BUILD_SHARED_LIBS)
    # on Windows 'SQLITE_BUILD_SHARED' and `CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS`
    # are needed to create '.lib' library in SHARED mode
    set(SQLITE_BUILD_SHARED TRUE)
    set(BUILD_SHARED_LIBS TRUE)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    add_library(${PROJECT_NAME} SHARED ${src_files})
    set_target_properties(${PROJECT_NAME} PROPERTIES
        OUTPUT_NAME   sqlite3
        PUBLIC_HEADER "${public_headers}"
        )

    target_include_directories(${PROJECT_NAME} PUBLIC ${include_dirs})
    target_compile_definitions(${PROJECT_NAME} PUBLIC ${user_options})

    # platform/compiler specific settings
    if(CMAKE_SYSTEM_NAME MATCHES Linux)
        find_package(Threads REQUIRED)
        target_link_libraries(${PROJECT_NAME} INTERFACE Threads::Threads ${CMAKE_DL_LIBS})
    elseif(WIN32 AND ${CMAKE_SIZEOF_VOID_P} LESS 8) # this is a 32bit windows
        option(BUILD_WITH_XPSDK "build for old 32bit (WinXP/2003) targets" OFF)
        if(BUILD_WITH_XPSDK)
            target_compile_definitions(${PROJECT_NAME} PUBLIC
                $<BUILD_INTERFACE:
                    -DSQLITE_OS_WINRT=0 -D_WIN32_WINNT=0x0502 -DWINVER=0x0502
                >
                )
        endif()
    endif()
endif()

# SQLite3 as STATIC library
if(SQLITE_BUILD_STATIC)
    add_library(${PROJECT_NAME}_static STATIC ${src_files})
    set_target_properties(${PROJECT_NAME}_static PROPERTIES
        OUTPUT_NAME   sqlite3_static
        PUBLIC_HEADER "${public_headers}"
        )

    target_include_directories(${PROJECT_NAME}_static PUBLIC ${include_dirs})
    target_compile_definitions(${PROJECT_NAME}_static PUBLIC ${user_options})

    # platform/compiler specific settings
    if(CMAKE_SYSTEM_NAME MATCHES Linux)
        find_package(Threads REQUIRED)
        target_link_libraries(${PROJECT_NAME}_static INTERFACE Threads::Threads ${CMAKE_DL_LIBS})
    elseif(WIN32 AND ${CMAKE_SIZEOF_VOID_P} LESS 8) # this is a 32bit windows
        option(BUILD_WITH_XPSDK "build for old 32bit (WinXP/2003) targets" OFF)
        if(BUILD_WITH_XPSDK)
            target_compile_definitions(${PROJECT_NAME}_static PUBLIC
                $<BUILD_INTERFACE:
                    -DSQLITE_OS_WINRT=0 -D_WIN32_WINNT=0x0502 -DWINVER=0x0502
                >
                )
        endif()
    endif()
endif()

#------------------------------------------------------------------------------
configure_file(sqlite3_config.h.in ${CMAKE_BINARY_DIR}/sqlite3_config.h)

if(SQLITE_BUILD_SHARED)
    list(APPEND install_targets ${PROJECT_NAME})
endif()

if(SQLITE_BUILD_STATIC)
    list(APPEND install_targets ${PROJECT_NAME}_static)
endif()

install(TARGETS ${install_targets} EXPORT ${PROJECT_NAME}Config
    ARCHIVE       DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY       DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME       DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
install(EXPORT  ${PROJECT_NAME}Config
    NAMESPACE   SQLite::
    DESTINATION cmake
    )
install(FILES
    ${CMAKE_BINARY_DIR}/sqlite3_config.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

#------------------------------------------------------------------------------
# SQLite3 shell application:
option(BUILD_SHELL "build SQLite3 shell application" OFF)
if(BUILD_SHELL)
    include(CMakeDependentOption)
    cmake_dependent_option(BUILD_SHELL_STATIC "build shell by static c/c++ runtime" ON "SQLITE_BUILD_STATIC" OFF)

    if(NOT SQLITE_BUILD_SHARED AND SQLITE_BUILD_STATIC AND NOT BUILD_SHELL_STATIC)
        message(FATAL_ERROR "No SQlite3 SHARED lib to be linked by SHARED shell application")
    endif()

    if(NOT BUILD_SHELL_STATIC)
        # add path to shared lib to RPATH so that you don't need to modify LD_LIBRARY_PATH 
        # when launching 'sqlite3' on UNIX platforms (must be before exe target is created)
        list(APPEND CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
    endif()

    add_executable(shell_app shell.c)
    set_target_properties(shell_app PROPERTIES OUTPUT_NAME sqlite3)
    if(BUILD_SHELL_STATIC)
        target_link_libraries(shell_app PRIVATE ${PROJECT_NAME}_static)
    else()
        target_link_libraries(shell_app PRIVATE ${PROJECT_NAME})
    endif()

    if(NOT WIN32 AND SQLITE_ENABLE_MATH_FUNCTIONS)
        # math (<math.h>) requires additional library to link 'libm'
        target_link_libraries(shell_app PRIVATE m)
    endif()
    if(UNIX)
        add_custom_command(
            TARGET shell_app POST_BUILD
            COMMAND "$<$<CONFIG:Release>:${CMAKE_STRIP}>" "$<$<CONFIG:Release>:$<TARGET_FILE:shell_app>>")
        if(CMAKE_BUILD_TYPE STREQUAL Release)
            add_custom_command(TARGET shell_app POST_BUILD
                COMMAND ${CMAKE_STRIP} sqlite3
                )
        endif()
    elseif(MSVC)
        foreach(flag CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_DEBUG)
            if(BUILD_SHELL_STATIC)
                string(REGEX REPLACE "/MD" "/MT" ${flag} "${${flag}}")
            else()
                string(REGEX REPLACE "/MT" "/MD" ${flag} "${${flag}}")
            endif()
            set(${flag} "${${flag}}" CACHE STRING "msvc flags" FORCE)
        endforeach()
    endif()
    install(TARGETS shell_app
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        )
endif()
