# Define VTK modules to link libf3d with
# Note that readers/importers specific modules
# should be listed here as reader instantiation is header only
set(F3D_VTK_MODULES
  VTK::CommonSystem
  VTK::IOImage
  VTK::InteractionWidgets
  f3d::VTKExtensionsApplicative
  f3d::VTKExtensionsCore
  f3d::VTKExtensionsReaders
  f3d::VTKExtensionsRendering
)

# Check modules
if(F3D_MODULE_RAYTRACING)
  if(TARGET VTK::RenderingRayTracing)
    message(STATUS "VTK::RenderingRayTracing module found")
    set(F3D_VTK_MODULES ${F3D_VTK_MODULES} VTK::RenderingRayTracing)
  else()
    message(FATAL_ERROR "F3D_MODULE_RAYTRACING option is enabled but VTK::RenderingRayTracing is not found")
  endif()
endif()

if(F3D_MODULE_EXTERNAL_RENDERING)
  if(TARGET VTK::RenderingExternal)
    message(STATUS "VTK::RenderingExternal module found")
    set(F3D_VTK_MODULES ${F3D_VTK_MODULES} VTK::RenderingExternal)
  else()
    message(FATAL_ERROR "F3D_MODULE_EXTERNAL_RENDERING option is enabled but VTK::RenderingExternal is not found")
  endif()
endif()

set(libf3d_compile_options_private "")
set(libf3d_compile_options_public "")
set(libf3d_link_options_public "")

# F3D_STRICT_BUILD
list(APPEND libf3d_compile_options_private ${f3d_strict_build_compile_options})

# Coverage
list(APPEND libf3d_compile_options_public ${f3d_coverage_compile_options})
list(APPEND libf3d_link_options_public ${f3d_coverage_link_options})

# Sanitizer
list(APPEND libf3d_compile_options_public ${f3d_sanitizer_compile_options})
list(APPEND libf3d_link_options_public ${f3d_sanitizer_link_options})

add_subdirectory(VTKExtensions)

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/src/config.cxx.in"
  "${CMAKE_CURRENT_BINARY_DIR}/src/config.cxx"
  @ONLY)

# Define libf3d target
set(F3D_SOURCE_FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/src/animationManager.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/camera_impl.cxx
  ${CMAKE_CURRENT_BINARY_DIR}/src/config.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/engine.cxx
  ${CMAKE_CURRENT_BINARY_DIR}/src/factory.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/image.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/init.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/interactor.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/interactor_impl.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/levenshtein.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/loader_impl.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/log.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/options.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/utils.cxx
  ${CMAKE_CURRENT_SOURCE_DIR}/src/window_impl.cxx
  )

# List of headers that will be installed
set(F3D_PUBLIC_HEADERS
  ${CMAKE_CURRENT_SOURCE_DIR}/plugin/reader.h
  ${CMAKE_CURRENT_SOURCE_DIR}/plugin/plugin.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/camera.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/engine.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/exception.h
  ${CMAKE_CURRENT_BINARY_DIR}/public/export.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/image.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/interactor.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/loader.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/log.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/options.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/types.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/utils.h
  ${CMAKE_CURRENT_SOURCE_DIR}/public/window.h
  )

add_library(libf3d
  ${F3D_SOURCE_FILES}
  )

target_include_directories(libf3d
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/public>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/public>
  PRIVATE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/private>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/private>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/plugin>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/plugin>
  )
if (F3D_USE_EXTERNAL_NLOHMANN_JSON)
  target_link_libraries(libf3d PRIVATE nlohmann_json::nlohmann_json)
else ()
  target_include_directories(libf3d PRIVATE
    $<BUILD_INTERFACE:${F3D_SOURCE_DIR}/external/nlohmann_json>)
endif ()

set_target_properties(libf3d PROPERTIES
  CXX_STANDARD 17
  CXX_VISIBILITY_PRESET hidden
  POSITION_INDEPENDENT_CODE ON
  OUTPUT_NAME "f3d"
  PDB_NAME "libf3d"
  )

# It can be useful to disable soversion in case the links are duplicated
# It happens with Python wheels for example
option(F3D_ENABLE_SOVERSION "Enable libf3d SOVERSION" ON)
mark_as_advanced(F3D_ENABLE_SOVERSION)
if(F3D_ENABLE_SOVERSION)
  set_target_properties(libf3d PROPERTIES
    VERSION ${F3D_MAJOR_VERSION}.${F3D_MINOR_VERSION}
    SOVERSION ${F3D_MAJOR_VERSION}
  )
endif()

# Generate export macros for exported public APIs
include(GenerateExportHeader)
option(F3D_EXCLUDE_DEPRECATED "Exclude deprecated functions of the library" OFF)
set(libf3d_no_deprecated)
if(F3D_EXCLUDE_DEPRECATED)
  set(libf3d_no_deprecated "DEFINE_NO_DEPRECATED")
endif()
generate_export_header(libf3d
  EXPORT_FILE_NAME public/export.h
  BASE_NAME F3D
  ${libf3d_no_deprecated})

target_include_directories(libf3d
  INTERFACE
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )

vtk_module_autoinit(TARGETS libf3d MODULES ${F3D_VTK_MODULES})

# The initialize function pointer of the static plugins must be added to the map of static plugins
# in the factory constructor. When loading a plugin, there's a lookup into this map.
get_property(F3D_STATIC_PLUGIN_TARGETS GLOBAL PROPERTY F3D_STATIC_PLUGINS)
set(F3D_STATIC_PLUGIN_MAP "${F3D_STATIC_PLUGIN_TARGETS}")
set(F3D_STATIC_PLUGIN_EXTERN "${F3D_STATIC_PLUGIN_TARGETS}")

list(TRANSFORM F3D_STATIC_PLUGIN_EXTERN REPLACE "^(.+)$" "extern f3d::plugin* init_plugin_static_\\1()\\\;")
list(JOIN F3D_STATIC_PLUGIN_EXTERN "\n" F3D_STATIC_PLUGIN_EXTERN)

list(TRANSFORM F3D_STATIC_PLUGIN_MAP REPLACE "^(.+)$" "this->StaticPluginInitializers[\"\\1\"] = init_plugin_static_\\1\\\;")
list(JOIN F3D_STATIC_PLUGIN_MAP "\n  " F3D_STATIC_PLUGIN_MAP)

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/src/factory.cxx.in"
  "${CMAKE_CURRENT_BINARY_DIR}/src/factory.cxx"
  @ONLY)

# libf3d links with the static plugins.
list(TRANSFORM F3D_STATIC_PLUGIN_TARGETS PREPEND "f3d-plugin-")

target_link_libraries(libf3d PRIVATE ${F3D_VTK_MODULES} ${F3D_STATIC_PLUGIN_TARGETS})
target_compile_options(libf3d PUBLIC ${libf3d_compile_options_public} PRIVATE ${libf3d_compile_options_private})
target_link_options(libf3d PUBLIC ${libf3d_link_options_public})

if(F3D_STRICT_BUILD AND MSVC)
  # There are warnings in VTK related to deprecated features in C++17
  target_compile_definitions(libf3d PRIVATE _SILENCE_ALL_CXX17_DEPRECATION_WARNINGS)
endif()

if (APPLE OR UNIX)
  # Make sure rpath is correctly set for libf3d
  get_target_property(target_type VTK::CommonCore TYPE)
  if (target_type STREQUAL SHARED_LIBRARY)
    set_target_properties(libf3d PROPERTIES INSTALL_RPATH "$<TARGET_FILE_DIR:VTK::CommonCore>")
  endif ()
endif ()

# Testing
if(BUILD_TESTING)
  add_subdirectory(testing)
endif()

# Installing
if(BUILD_SHARED_LIBS)
  install(TARGETS libf3d
    EXPORT f3dTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT library
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR} COMPONENT library
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT library)

  install(FILES ${F3D_PUBLIC_HEADERS}
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/f3d"
    COMPONENT sdk
    EXCLUDE_FROM_ALL)

  install(EXPORT f3dTargets
    NAMESPACE f3d::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/f3d"
    COMPONENT sdk
    EXCLUDE_FROM_ALL)

  include(CMakePackageConfigHelpers)
  configure_package_config_file(
    "${F3D_SOURCE_DIR}/cmake/f3dConfig.cmake.in" "${CMAKE_BINARY_DIR}/cmake/f3dConfig.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/f3d")

  write_basic_package_version_file(
    "${CMAKE_BINARY_DIR}/cmake/f3dConfigVersion.cmake"
    VERSION "${F3D_VERSION}"
    COMPATIBILITY SameMinorVersion)

  install(
    FILES
      "${CMAKE_BINARY_DIR}/cmake/f3dConfig.cmake"
      "${CMAKE_BINARY_DIR}/cmake/f3dConfigVersion.cmake"
      "${F3D_SOURCE_DIR}/cmake/f3dEmbed.cmake"
      "${F3D_SOURCE_DIR}/cmake/f3dPlugin.cmake"
      "${F3D_SOURCE_DIR}/cmake/plugin.cxx.in"
      "${F3D_SOURCE_DIR}/cmake/plugin.desktop.in"
      "${F3D_SOURCE_DIR}/cmake/plugin.thumbnailer.in"
      "${F3D_SOURCE_DIR}/cmake/readerBoilerPlate.h.in"
    DESTINATION
      "${CMAKE_INSTALL_LIBDIR}/cmake/f3d"
    COMPONENT sdk
    EXCLUDE_FROM_ALL)
endif()
