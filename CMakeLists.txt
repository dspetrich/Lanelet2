cmake_minimum_required(VERSION 3.16)
project(lanelet2 VERSION 1.2.2 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
cmake_policy(SET CMP0167 NEW)

# ─────────────────────────────────────────────────────────────────────────────
# Dependencies
# ─────────────────────────────────────────────────────────────────────────────
find_package(Eigen3 REQUIRED NO_MODULE)
find_package(Boost REQUIRED CONFIG COMPONENTS filesystem serialization)

# pugixml — prefer Homebrew install
find_package(pugixml REQUIRED)

# GeographicLib — prefer Homebrew install
find_package(GeographicLib REQUIRED)

# ─────────────────────────────────────────────────────────────────────────────
# lanelet2_core
# ─────────────────────────────────────────────────────────────────────────────
file(GLOB LANELET2_CORE_SRC "lanelet2_core/src/*.cpp")
add_library(lanelet2_core STATIC ${LANELET2_CORE_SRC})
target_include_directories(lanelet2_core PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_core/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(lanelet2_core PUBLIC
    Eigen3::Eigen
    Boost::headers
    Boost::filesystem
    Boost::serialization
)

# ─────────────────────────────────────────────────────────────────────────────
# lanelet2_io
# ─────────────────────────────────────────────────────────────────────────────
file(GLOB LANELET2_IO_SRC "lanelet2_io/src/*.cpp")
add_library(lanelet2_io STATIC ${LANELET2_IO_SRC})
target_include_directories(lanelet2_io PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_io/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(lanelet2_io PUBLIC
    lanelet2_core
    pugixml::pugixml
    Boost::filesystem
)

# ─────────────────────────────────────────────────────────────────────────────
# lanelet2_projection
# ─────────────────────────────────────────────────────────────────────────────
file(GLOB LANELET2_PROJECTION_SRC "lanelet2_projection/src/*.cpp")
add_library(lanelet2_projection STATIC ${LANELET2_PROJECTION_SRC})
target_include_directories(lanelet2_projection PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_projection/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(lanelet2_projection PUBLIC
    lanelet2_core
    lanelet2_io
    ${GeographicLib_LIBRARIES}
)
target_include_directories(lanelet2_projection PUBLIC ${GeographicLib_INCLUDE_DIRS})

# ─────────────────────────────────────────────────────────────────────────────
# lanelet2_traffic_rules
# ─────────────────────────────────────────────────────────────────────────────
file(GLOB LANELET2_TRAFFIC_SRC "lanelet2_traffic_rules/src/*.cpp")
add_library(lanelet2_traffic_rules STATIC ${LANELET2_TRAFFIC_SRC})
target_include_directories(lanelet2_traffic_rules PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_traffic_rules/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(lanelet2_traffic_rules PUBLIC
    lanelet2_core
)

# ─────────────────────────────────────────────────────────────────────────────
# lanelet2_routing
# ─────────────────────────────────────────────────────────────────────────────
file(GLOB LANELET2_ROUTING_SRC "lanelet2_routing/src/*.cpp")
add_library(lanelet2_routing STATIC ${LANELET2_ROUTING_SRC})
target_include_directories(lanelet2_routing PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/lanelet2_routing/include>
    $<INSTALL_INTERFACE:include>
)
target_link_libraries(lanelet2_routing PUBLIC
    lanelet2_core
    lanelet2_traffic_rules
    Boost::headers
)

# ─────────────────────────────────────────────────────────────────────────────
# Install
# ─────────────────────────────────────────────────────────────────────────────
include(GNUInstallDirs)
install(TARGETS
    lanelet2_core lanelet2_io lanelet2_projection
    lanelet2_traffic_rules lanelet2_routing
    EXPORT lanelet2Targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
foreach(pkg lanelet2_core lanelet2_io lanelet2_projection lanelet2_traffic_rules lanelet2_routing)
    install(DIRECTORY ${pkg}/include/
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
endforeach()

# Export CMake targets so find_package(lanelet2_core) works
install(EXPORT lanelet2Targets
    FILE lanelet2Targets.cmake
    NAMESPACE lanelet2::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/lanelet2
)

# Generate per-package Config files
foreach(pkg lanelet2_core lanelet2_io lanelet2_projection lanelet2_traffic_rules lanelet2_routing)
    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${pkg}Config.cmake"
"include(CMakeFindDependencyMacro)
find_dependency(Eigen3)
find_dependency(Boost COMPONENTS filesystem serialization)
include(\"\${CMAKE_CURRENT_LIST_DIR}/../lanelet2/lanelet2Targets.cmake\" OPTIONAL)
# Direct target alias for find_package(${pkg})
if(NOT TARGET ${pkg})
    add_library(${pkg} ALIAS lanelet2::${pkg})
endif()
")
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${pkg}Config.cmake"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${pkg}"
    )
endforeach()
