cmake_minimum_required(VERSION 3.18)
project(KaniDivert LANGUAGES C)

include(FetchContent)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(INC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

# =============== object ===============
set(DLL_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dll)
set(DLL_SRC_FILES ${DLL_SRC_DIR}/windivert.c)

add_library(KaniDivert_obj OBJECT ${DLL_SRC_FILES})
target_include_directories(KaniDivert_obj PUBLIC ${INC_DIR})
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
    set(CMAKE_C_FLAGS_RELEASE "")
    target_compile_options(KaniDivert_obj PRIVATE
            /W3 /GS- /WX- /GR- /EHs-c-
            $<$<CONFIG:Debug>: /Od /Zi>
            $<$<CONFIG:Release>: /Os /Oi /GL /Gy /GF /fp:fast /Ob2>
    )
    target_link_options(KaniDivert_obj PRIVATE
            $<$<CONFIG:Release>:/LTCG>
    )
elseif(MINGW)
    set(CMAKE_C_FLAGS_DEBUG "-Og -g")
    set(CMAKE_C_FLAGS_RELEASE "")
    target_compile_options(KaniDivert_obj PRIVATE
            #[[-fno-ident -Wall -Wno-pointer-to-int-cast -fno-stack-protector -ffast-math -Os]]
            -fno-ident -Wall -Wno-pointer-to-int-cast -fno-stack-protector
            $<$<CONFIG:Debug>: -Og -g>
            $<$<CONFIG:Release>: -Os -ffast-math>
    )
else()
    message(FATAL_ERROR "[KaniDivert] Unsupported compiler. Only MSVC and MinGW are supported.")
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
        $<BUILD_INTERFACE:${INC_DIR}>
        $<INSTALL_INTERFACE:include>
)
target_compile_definitions(KaniDivert_static PUBLIC
        WINDIVERTSTATIC
)
add_library(KaniDivert::static ALIAS KaniDivert_static)
# ====================================== static


# =============== shared ===============
set(DLL_ENTRY_POINT "WinDivertDllEntry")
set(DLL_EXPORT_DEF_FILE ${CMAKE_CURRENT_SOURCE_DIR}/dll/windivert.def)

add_library(KaniDivert_shared SHARED $<TARGET_OBJECTS:KaniDivert_obj>)
set_target_properties(KaniDivert_shared PROPERTIES
        POSITION_INDEPENDENT_CODE ON
        PREFIX ""
        OUTPUT_NAME "WinDivert"
)
target_include_directories(KaniDivert_shared PUBLIC
        $<BUILD_INTERFACE:${INC_DIR}>
        $<INSTALL_INTERFACE:include>
)
target_compile_definitions(KaniDivert_shared PRIVATE
        _USRDLL
        DLL_EXPORTS
        WINDIVERTEXPORTS
)
add_library(KaniDivert::shared ALIAS KaniDivert_shared)

if(MSVC)
    target_sources(KaniDivert_shared PRIVATE ${DLL_EXPORT_DEF_FILE})
    target_link_options(KaniDivert_shared PRIVATE
            /ENTRY:${DLL_ENTRY_POINT}
            /NODEFAULTLIB
            /SUBSYSTEM:WINDOWS
            /MANIFEST:NO
            /INCREMENTAL:NO
            /OPT:REF
            /OPT:ICF
    )
elseif(MINGW)
    target_link_options(KaniDivert_shared PRIVATE
            -shared
            -Wl,--enable-stdcall-fixup
            -Wl,--entry=${DLL_ENTRY_POINT}
            -nostdlib
    )
    target_link_libraries(KaniDivert_shared kernel32 advapi32)

    find_program(DLLTOOL_EXECUTABLE NAMES ${CMAKE_C_COMPILER_TARGET}-dlltool dlltool)
    if(DLLTOOL_EXECUTABLE)
        add_custom_command(TARGET KaniDivert_shared POST_BUILD
                COMMAND ${DLLTOOL_EXECUTABLE}
                --dllname $<TARGET_FILE_NAME:KaniDivert_shared>
                --def ${DLL_EXPORT_DEF_FILE}
                --output-lib $<TARGET_FILE_DIR:KaniDivert_shared>/WinDivert.lib
                COMMENT "Generating import library"
        )
    else()
        message(FATAL_ERROR "[KaniDivert] dlltool not found — required for MinGW builds.")
    endif()

    if(CMAKE_BUILD_TYPE MATCHES "^[Rr][Ee][Ll][Ee][Aa][Ss][Ee]$")
        find_program(STRIP_EXECUTABLE NAMES ${CMAKE_C_COMPILER_TARGET}-strip strip)
        if(STRIP_EXECUTABLE)
            add_custom_command(TARGET KaniDivert_shared POST_BUILD
                    COMMAND ${STRIP_EXECUTABLE} $<TARGET_FILE:KaniDivert_shared>
                    COMMENT "Stripping symbols from DLL"
            )
        else()
            message(WARNING "[KaniDivert] strip not found — symbols will not be stripped from the DLL in Release build.")
        endif()
    endif()
endif()
# ====================================== shared


# =============== driver ===============
FetchContent_Declare(
        FindWDK
        GIT_REPOSITORY https://github.com/SergiusTheBest/FindWDK.git
)
FetchContent_MakeAvailable(FindWDK)
list(APPEND CMAKE_MODULE_PATH "${findwdk_SOURCE_DIR}/cmake")
find_package(WDK)

if (WDK_FOUND)
    set(SYS_SRC_DIR     ${CMAKE_CURRENT_SOURCE_DIR}/sys)
    set(SYS_GEN_DIR     ${CMAKE_BINARY_DIR}/windivert_generated)
    file(MAKE_DIRECTORY ${SYS_GEN_DIR})

    set(SYS_MC_FILE     ${SYS_SRC_DIR}/windivert_log.mc)
    set(SYS_MC_HDR_FILE ${SYS_GEN_DIR}/windivert_log.h)
    set(SYS_MC_RC_FILE  ${SYS_GEN_DIR}/windivert_log.rc)
    add_custom_command(
            OUTPUT ${SYS_MC_HDR_FILE} ${SYS_MC_RC_FILE}
            COMMAND mc.exe -h ${SYS_GEN_DIR} -r ${SYS_GEN_DIR} ${SYS_MC_FILE}
            DEPENDS ${SYS_MC_FILE}
            COMMENT "Generating message header and resource file"
    )

    set(SYS_SRC_FILES
            ${SYS_SRC_DIR}/windivert.c
            ${SYS_MC_HDR_FILE}
    )

    wdk_add_driver(KaniDivert_driver KMDF 1.15 ${SYS_SRC_FILES})
    target_sources(KaniDivert_driver PRIVATE ${SYS_SRC_DIR}/windivert.rc)
    target_include_directories(KaniDivert_driver PRIVATE
            ${INC_DIR} ${DLL_SRC_DIR} ${SYS_GEN_DIR}
    )
    target_link_libraries(KaniDivert_driver
            WDK::NDIS WDK::FWPKCLNT WDK::WDMSEC uuid
    )
    target_compile_definitions(KaniDivert_driver PRIVATE
            UNICODE _UNICODE
            NDIS60 NDIS_SUPPORT_NDIS60
            BINARY_COMPATIBLE=0
            NT
    )

    if(${WDK_PLATFORM} STREQUAL "x64")
        set_target_properties(KaniDivert_driver PROPERTIES OUTPUT_NAME "WinDivert64")
    elseif(${WDK_PLATFORM} STREQUAL "x86")
        set_target_properties(KaniDivert_driver PROPERTIES OUTPUT_NAME "WinDivert32")
    endif()
else()
    message(STATUS "[KaniDivert] WDK not found - skipping driver build")
endif()
# ====================================== driver


function(download_official_sys target_proj destination_dir)
    if(NOT destination_dir)
        message(FATAL_ERROR "[WinDivert::download_official_sys] destination_dir is required")
    endif()

    FetchContent_Declare(
            windivert_official
            URL https://github.com/basil00/WinDivert/releases/download/v2.2.2/WinDivert-2.2.2-A.zip
            DOWNLOAD_EXTRACT_TIMESTAMP true
    )
    FetchContent_MakeAvailable(windivert_official)

    file(MAKE_DIRECTORY "${destination_dir}")
    add_custom_command(TARGET ${target_proj} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy
            "${windivert_official_SOURCE_DIR}/x64/WinDivert64.sys"
            "${destination_dir}/WinDivert64.sys"
            COMMAND ${CMAKE_COMMAND} -E copy
            "${windivert_official_SOURCE_DIR}/x86/WinDivert32.sys"
            "${destination_dir}/WinDivert32.sys"
    )

    message(STATUS "[WinDivert::download_official_sys] Will copy .sys files to ${destination_dir}")
endfunction()