cmake_minimum_required(VERSION 3.16)
project(lanelet2 VERSION 1.2.2 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
cmake_policy(SET CMP0167 NEW)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

find_package(Eigen3 REQUIRED NO_MODULE)
find_package(Boost REQUIRED CONFIG COMPONENTS filesystem serialization)
find_package(pugixml REQUIRED)
find_package(GeographicLib REQUIRED)

set(LANELET2_COMMON_INCLUDE_DIRS
  ${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_core/include
  ${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_io/include
  ${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_traffic_rules/include
  ${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_projection/include
  ${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_routing/include
)

function(add_lanelet2_library target)
  cmake_parse_arguments(ARG "" "" "SOURCES;LINK_LIBS" ${ARGN})
  add_library(${target} STATIC ${ARG_SOURCES})
  target_include_directories(${target} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${target}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  )
  target_link_libraries(${target} PUBLIC ${ARG_LINK_LIBS})
  target_compile_definitions(${target} PUBLIC BOOST_ALLOW_DEPRECATED_HEADERS)

  install(TARGETS ${target}
    EXPORT ${target}Targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )
  install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/${target}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )
  install(EXPORT ${target}Targets
    FILE ${target}Targets.cmake
    NAMESPACE ""
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${target}
  )
endfunction()

add_lanelet2_library(lanelet2_core
  SOURCES
    lanelet2_core/src/Attribute.cpp
    lanelet2_core/src/BasicRegulatoryElements.cpp
    lanelet2_core/src/Lanelet.cpp
    lanelet2_core/src/LaneletMap.cpp
    lanelet2_core/src/LaneletSequence.cpp
    lanelet2_core/src/LineStringGeometry.cpp
    lanelet2_core/src/PolygonTriangulationGeometry.cpp
    lanelet2_core/src/RegulatoryElement.cpp
    lanelet2_core/src/RegulatoryElementGeometry.cpp
  LINK_LIBS Eigen3::Eigen Boost::headers
)

add_lanelet2_library(lanelet2_io
  SOURCES
    lanelet2_io/src/BinHandler.cpp
    lanelet2_io/src/Factory.cpp
    lanelet2_io/src/Io.cpp
    lanelet2_io/src/OsmFile.cpp
    lanelet2_io/src/OsmHandlerLoad.cpp
    lanelet2_io/src/OsmHandlerWrite.cpp
  LINK_LIBS lanelet2_core pugixml::pugixml Boost::headers Boost::filesystem Boost::serialization
)

add_lanelet2_library(lanelet2_traffic_rules
  SOURCES
    lanelet2_traffic_rules/src/GenericTrafficRules.cpp
    lanelet2_traffic_rules/src/GermanTrafficRules.cpp
    lanelet2_traffic_rules/src/TrafficRulesFactory.cpp
  LINK_LIBS lanelet2_core
)

add_lanelet2_library(lanelet2_projection
  SOURCES
    lanelet2_projection/src/Geocentric.cpp
    lanelet2_projection/src/LocalCartesian.cpp
    lanelet2_projection/src/UTM.cpp
  LINK_LIBS lanelet2_io GeographicLib::GeographicLib
)

add_lanelet2_library(lanelet2_routing
  SOURCES
    lanelet2_routing/src/LaneletPath.cpp
    lanelet2_routing/src/Route.cpp
    lanelet2_routing/src/RouteBuilder.cpp
    lanelet2_routing/src/RoutingCost.cpp
    lanelet2_routing/src/RoutingGraph.cpp
    lanelet2_routing/src/RoutingGraphBuilder.cpp
  LINK_LIBS lanelet2_core lanelet2_traffic_rules Boost::headers
)

function(write_lanelet2_config target dependencies)
  set(config_file ${CMAKE_CURRENT_BINARY_DIR}/${target}Config.cmake)
  file(WRITE ${config_file} "include(CMakeFindDependencyMacro)\n")
  foreach(dep IN LISTS ${dependencies})
    if(target STREQUAL "lanelet2_io" AND dep STREQUAL "Boost")
      file(APPEND ${config_file} "find_dependency(Boost COMPONENTS filesystem serialization)\n")
    else()
      file(APPEND ${config_file} "find_dependency(${dep})\n")
    endif()
  endforeach()
  file(APPEND ${config_file} "include(\"\${CMAKE_CURRENT_LIST_DIR}/${target}Targets.cmake\")\n")
  write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/${target}ConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
  )
  install(FILES
    ${config_file}
    ${CMAKE_CURRENT_BINARY_DIR}/${target}ConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${target}
  )
endfunction()

set(core_deps Eigen3 Boost)
write_lanelet2_config(lanelet2_core core_deps)
set(io_deps lanelet2_core pugixml Boost)
write_lanelet2_config(lanelet2_io io_deps)
set(traffic_deps lanelet2_core)
write_lanelet2_config(lanelet2_traffic_rules traffic_deps)
set(projection_deps lanelet2_io GeographicLib)
write_lanelet2_config(lanelet2_projection projection_deps)
set(routing_deps lanelet2_core lanelet2_traffic_rules Boost)
write_lanelet2_config(lanelet2_routing routing_deps)
