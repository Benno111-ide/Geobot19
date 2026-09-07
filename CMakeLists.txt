cmake_minimum_required(VERSION 3.25)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

if (APPLE)
    if (CMAKE_SYSTEM_NAME STREQUAL "iOS" OR GEODE_TARGET_PLATFORM STREQUAL "iOS")
        # Let Geode's iOS toolchain keep a single device architecture.
        set(CMAKE_OSX_ARCHITECTURES "arm64")
    else()
        set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64")
    endif()
endif()

project(geobot VERSION 2.0.0)
option(geobot_FAST_BUILD "Optimize build settings for faster compile times" ON)
option(geobot_ENABLE_FRAMEPERFECT_DETECTION "Build frameperfect detection support" OFF)
option(geobot_ENABLE_PATHFINDER "Build pathfinder support" OFF)

option(geobot_ENABLE_TEST_BUILDS "Enable test targets in this project and dependencies" OFF)
set(BUILD_TESTING ${geobot_ENABLE_TEST_BUILDS} CACHE BOOL "Build tests" FORCE)
 
add_library(${PROJECT_NAME} SHARED
    src/main.cpp
    src/global.cpp
    src/macro.cpp
    src/keybinds.cpp
    src/utils/utils.cpp
    src/gdr/gdr.cpp
    src/renderer/renderer.cpp

    src/hacks/other.cpp
    src/hacks/tps_bypass.cpp
    src/hacks/frame_stepper.cpp
    src/hacks/layout_mode.cpp
    src/hacks/show_trajectory.cpp
    src/hacks/coin_finder.cpp
    src/hacks/clickbot.cpp
    src/hacks/autoclicker.cpp

    src/ui/macro_editor.cpp
    src/ui/button_edit_layer.cpp
    src/ui/game_ui.cpp
    src/ui/record_layer.cpp
    src/ui/render_settings_layer.cpp
    src/ui/load_macro_layer.cpp
    src/ui/clickbot_layer.cpp
    src/ui/button_setting.cpp

    src/practice_fixes/input.cpp
    src/practice_fixes/player.cpp
    src/practice_fixes/play_layer.cpp
)

if (NOT DEFINED ENV{GEODE_SDK})
    message(FATAL_ERROR "Unable to find Geode SDK! Please define GEODE_SDK environment variable to point to Geode")
else()
    message(STATUS "Found Geode: $ENV{GEODE_SDK}")
endif()

# This branch targets the loader bundled with 19gdps.com u9, not upstream Geode.
file(READ "$ENV{GEODE_SDK}/VERSION" _geobot_sdk_version)
string(STRIP "${_geobot_sdk_version}" _geobot_sdk_version)
if (NOT _geobot_sdk_version STREQUAL "4.9.0")
    message(FATAL_ERROR "Use qimiko/geode's gdps SDK (Geode 4.9.0); see docs/backport-1.9.md")
endif()
set(BINDINGS_VERSIONED_ONLY ON CACHE BOOL "Use the 1.920 enums" FORCE)
set(GEODE_GD_VERSION "1.920")
set(GEODE_COMP_GD_VERSION 19200)
add_subdirectory($ENV{GEODE_SDK} ${CMAKE_CURRENT_BINARY_DIR}/geode)
if (NOT GEODE_TARGET_PLATFORM MATCHES "^(Win32|Android32)$")
    message(FATAL_ERROR "1.9 GDPS supports Win32 and Android32 only")
endif()
if (NOT CMAKE_SIZEOF_VOID_P EQUAL 4)
    message(FATAL_ERROR "1.9 GDPS requires a 32-bit toolchain (Windows: -A Win32)")
endif()
target_compile_definitions(${PROJECT_NAME} PRIVATE
    GEOBOT_ENABLE_FRAMEPERFECT_DETECTION=$<BOOL:${geobot_ENABLE_FRAMEPERFECT_DETECTION}>
    GEOBOT_ENABLE_PATHFINDER=$<BOOL:${geobot_ENABLE_PATHFINDER}>
)

set(_geobot_mod_source_dir "${CMAKE_CURRENT_BINARY_DIR}/geobot-mod-source")
file(MAKE_DIRECTORY "${_geobot_mod_source_dir}")

file(READ "${CMAKE_CURRENT_SOURCE_DIR}/mod.json" _geobot_mod_json)
if (NOT geobot_ENABLE_FRAMEPERFECT_DETECTION)
    string(JSON _geobot_mod_json REMOVE "${_geobot_mod_json}"
        "settings" "feature_flag_frameperfect_detection"
    )
endif()
if (NOT geobot_ENABLE_PATHFINDER)
    string(JSON _geobot_mod_json REMOVE "${_geobot_mod_json}"
        "settings" "feature_flag_pathfinder"
    )
    string(JSON _geobot_mod_json REMOVE "${_geobot_mod_json}"
        "settings" "pathfinder_solver_executable"
    )
    string(JSON _geobot_mod_json REMOVE "${_geobot_mod_json}"
        "settings" "pathfinder_solver_args"
    )
    string(JSON _geobot_mod_json REMOVE "${_geobot_mod_json}"
        "settings" "pathfinder_output_folder"
    )
endif()
file(WRITE "${_geobot_mod_source_dir}/mod.json" "${_geobot_mod_json}\n")

foreach(_geobot_mod_file
    about.md
    changelog.md
    support.md
    logo.png
    resources/GJ_commentSide2_001_White.png
    resources/GJ_commentTop2_001_White.png
    resources/default_hold_click.mp3
    resources/default_release_click.mp3
    resources/default_hold_left.mp3
    resources/default_release_left.mp3
    resources/default_hold_right.mp3
    resources/default_release_right.mp3
)
    get_filename_component(_geobot_mod_file_dir "${_geobot_mod_file}" DIRECTORY)
    if (NOT _geobot_mod_file_dir STREQUAL "")
        file(MAKE_DIRECTORY "${_geobot_mod_source_dir}/${_geobot_mod_file_dir}")
    endif()
    configure_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/${_geobot_mod_file}"
        "${_geobot_mod_source_dir}/${_geobot_mod_file}"
        COPYONLY
    )
endforeach()

# setup_geode_mod adds post-build commands to the mod target. CMake requires
# TARGET-style add_custom_command calls to run in the directory that created
# the target, so invoke the helper here rather than from a generated subdirectory.
# Point it at the filtered package sources while it discovers mod.json/assets.
set(_geobot_source_dir "${CMAKE_CURRENT_SOURCE_DIR}")
set(CMAKE_CURRENT_SOURCE_DIR "${_geobot_mod_source_dir}")
setup_geode_mod(${PROJECT_NAME})
set(CMAKE_CURRENT_SOURCE_DIR "${_geobot_source_dir}")

if (geobot_FAST_BUILD AND TARGET ${PROJECT_NAME})
    # Faster builds: disable LTO. Unity build is disabled because Geode's
    # macro-generated symbols ($modify / $execute) can collide in unity TUs.
    set_property(TARGET ${PROJECT_NAME} PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        UNITY_BUILD OFF
    )
endif()
