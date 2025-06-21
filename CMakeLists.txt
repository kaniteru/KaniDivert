cmake_minimum_required(VERSION 3.13)
project(KaniDivert LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(ENTRY_POINT "WinDivertDllEntry")
set(HDR_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)
set(SRC_FILES src/windivert.c)
set(EXPORT_DEF_FILE ${CMAKE_CURRENT_SOURCE_DIR}/src/windivert.def)


# =============== object ===============
add_library(KaniDivert_obj OBJECT ${SRC_FILES})
target_include_directories(KaniDivert_obj PUBLIC ${HDR_DIR})
set_target_properties(KaniDivert_obj PROPERTIES POSITION_INDEPENDENT_CODE ON)

target_compile_definitions(KaniDivert_obj PRIVATE
        WIN32
        _WINDOWS
        $<$<CONFIG:Debug>:DEBUG;_DEBUG>
        $<$<CONFIG:Release>:NDEBUG>
)

if(MSVC)
    string(REPLACE "/RTC1" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    string(REPLACE "/RTCs" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    string(REPLACE "/RTCu" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
    target_compile_options(KaniDivert_obj PRIVATE
            /W3 /Os /Oi /GS- /WX- /GR- /EHs-c- /fp:fast
    )
elseif(MINGW)
    target_compile_options(KaniDivert_obj PRIVATE
            -fno-ident -Wall -Wno-pointer-to-int-cast -Os
    )
else()
    message(FATAL_ERROR "Unsupported compiler. Only MSVC and MinGW are supported.")
endif()
# ====================================== object


# =============== static ===============
add_library(KaniDivert_static STATIC $<TARGET_OBJECTS:KaniDivert_obj>)
set_target_properties(KaniDivert_static PROPERTIES
        POSITION_INDEPENDENT_CODE ON
        PREFIX ""
        OUTPUT_NAME "WinDivert_static"
)
target_include_directories(KaniDivert_static PUBLIC
        $<BUILD_INTERFACE:${HDR_DIR}>
        $<INSTALL_INTERFACE:include>
)
target_compile_definitions(KaniDivert_static PUBLIC
        WINDIVERTSTATIC
)
add_library(KaniDivert::static ALIAS KaniDivert_static)
# ====================================== static


# =============== shared ===============
add_library(KaniDivert_shared SHARED $<TARGET_OBJECTS:KaniDivert_obj>)
set_target_properties(KaniDivert_shared PROPERTIES
        PREFIX ""
        OUTPUT_NAME "WinDivert"
)
target_include_directories(KaniDivert_shared PUBLIC
        $<BUILD_INTERFACE:${HDR_DIR}>
        $<INSTALL_INTERFACE:include>
)
target_compile_definitions(KaniDivert_shared PRIVATE
        _USRDLL
        DLL_EXPORTS
        WINDIVERTEXPORTS
)
add_library(KaniDivert::shared ALIAS KaniDivert_shared)

if(MSVC)
    target_sources(KaniDivert_shared PRIVATE
            ${EXPORT_DEF_FILE}
    )

    set_target_properties(KaniDivert_shared PROPERTIES
            LINK_FLAGS "/ENTRY:${ENTRY_POINT} /NODEFAULTLIB /SUBSYSTEM:WINDOWS"
    )

    target_link_options(KaniDivert_shared PRIVATE
            /MANIFEST:NO
            /INCREMENTAL:NO
            /OPT:REF
            /OPT:ICF
    )
elseif(MINGW)
    target_link_options(KaniDivert_shared PRIVATE
            -shared
            -Wl,--enable-stdcall-fixup
            -Wl,--entry=${ENTRY_POINT}
            -nostdlib
    )

    target_link_libraries(KaniDivert_shared
            kernel32 advapi32
    )

    find_program(DLLTOOL_EXECUTABLE NAMES ${CMAKE_C_COMPILER_TARGET}-dlltool dlltool)
    if(DLLTOOL_EXECUTABLE)
        add_custom_command(TARGET KaniDivert_shared POST_BUILD
                COMMAND ${DLLTOOL_EXECUTABLE}
                --dllname $<TARGET_FILE_NAME:KaniDivert_shared>
                --def ${EXPORT_DEF_FILE}
                --output-lib $<TARGET_FILE_DIR:KaniDivert_shared>/WinDivert.lib
                COMMENT "Generating import library"
        )
    else()
        message(FATAL_ERROR "dlltool not found — required for MinGW builds.")
    endif()

    if(CMAKE_BUILD_TYPE MATCHES "^[Rr][Ee][Ll][Ee][Aa][Ss][Ee]$")
        find_program(STRIP_EXECUTABLE NAMES ${CMAKE_C_COMPILER_TARGET}-strip strip)
        if(STRIP_EXECUTABLE)
            add_custom_command(TARGET KaniDivert_shared POST_BUILD
                    COMMAND ${STRIP_EXECUTABLE} $<TARGET_FILE:KaniDivert_shared>
                    COMMENT "Stripping symbols from DLL"
            )
        else()
            message(WARNING "strip not found — symbols will not be stripped from the DLL in Release build.")
        endif()
    endif()
endif()
# ====================================== shared