cmake_minimum_required(VERSION 3.20)

# 强制 64 位构建检查
if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    message(FATAL_ERROR "32-bit builds are not supported. Please use 64-bit toolchain.")
endif()

# 设置静态链接目标
set(VCPKG_TARGET_TRIPLET "x64-windows-static" CACHE STRING "强制使用静态库版本")

# Vcpkg 工具链自动配置
set(VCPKG_MANIFEST_MODE OFF) # 禁用清单模式，使用全局依赖
if(NOT CMAKE_TOOLCHAIN_FILE)
    if(DEFINED ENV{VCPKG_ROOT})
        set(vcpkg_toolchain "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")
        if(EXISTS "${vcpkg_toolchain}")
            set(CMAKE_TOOLCHAIN_FILE "${vcpkg_toolchain}"
                CACHE STRING "Vcpkg toolchain file")
            message(STATUS "Using vcpkg from: $ENV{VCPKG_ROOT}")
        else()
            message(WARNING "vcpkg toolchain file not found at: ${vcpkg_toolchain}")
        endif()
    else()
        message(STATUS "VCPKG_ROOT not set, skipping vcpkg toolchain setup")
    endif()
else()
    message(STATUS "Using custom toolchain: ${CMAKE_TOOLCHAIN_FILE}")
endif()

# 优先使用 Ninja 生成器
find_program(NINJA_EXECUTABLE NAMES ninja)
if(NINJA_EXECUTABLE)
    set(CMAKE_GENERATOR "Ninja" CACHE INTERNAL "Use Ninja if available")
endif()

project(lt_make_torrent
    VERSION 1.0
    LANGUAGES CXX
    DESCRIPTION "libtorrent make torrent"
)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 全局静态链接配置
set(BUILD_SHARED_LIBS OFF)
if(MSVC)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# 设置 vcpkg 的搜索路径
if(CMAKE_TOOLCHAIN_FILE)
    # 获取 vcpkg 安装路径
    get_filename_component(VCPKG_ROOT ${CMAKE_TOOLCHAIN_FILE} DIRECTORY)
    get_filename_component(VCPKG_ROOT ${VCPKG_ROOT} DIRECTORY)

    # 设置库和包含路径
    set(CMAKE_PREFIX_PATH
        "${VCPKG_ROOT}/installed/${VCPKG_TARGET_TRIPLET}"
        ${CMAKE_PREFIX_PATH}
    )

    message(STATUS "Vcpkg installed directory: ${VCPKG_ROOT}/installed/${VCPKG_TARGET_TRIPLET}")
endif()

# 查找 libtorrent
find_package(LibtorrentRasterbar REQUIRED)
if(TARGET LibtorrentRasterbar::torrent-rasterbar)
    message(STATUS "Found libtorrent: ${LibtorrentRasterbar_VERSION}")
elseif(TARGET LibtorrentRasterbar::torrent)
    message(STATUS "Found libtorrent (alternative target)")
else()
    # 手动设置路径
    find_path(LIBTORRENT_INCLUDE_DIR libtorrent/torrent.hpp
        PATHS "${VCPKG_ROOT}/installed/${VCPKG_TARGET_TRIPLET}/include"
    )

    find_library(LIBTORRENT_LIBRARY
        NAMES torrent-rasterbar.lib libtorrent-rasterbar.a
        PATHS "${VCPKG_ROOT}/installed/${VCPKG_TARGET_TRIPLET}/lib"
    )

    if(LIBTORRENT_INCLUDE_DIR AND LIBTORRENT_LIBRARY)
        message(STATUS "Found libtorrent manually: ${LIBTORRENT_LIBRARY}")
        add_library(LibtorrentRasterbar::torrent INTERFACE IMPORTED)
        set_target_properties(LibtorrentRasterbar::torrent PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES "${LIBTORRENT_INCLUDE_DIR}"
            INTERFACE_LINK_LIBRARIES "${LIBTORRENT_LIBRARY}"
        )
    else()
        message(FATAL_ERROR "libtorrent not found! Please install: vcpkg install libtorrent:x64-windows-static")
    endif()
endif()

# 使用更安全的文件收集方式
set(SOURCES_DIR src)
file(GLOB SOURCES CONFIGURE_DEPENDS "${SOURCES_DIR}/*.cpp")
file(GLOB HEADERS CONFIGURE_DEPENDS "${SOURCES_DIR}/*.hpp")

# 添加可执行目标
add_executable(${PROJECT_NAME} ${SOURCES} ${HEADERS})

# 链接 libtorrent
if(TARGET LibtorrentRasterbar::torrent-rasterbar)
    target_link_libraries(${PROJECT_NAME} PRIVATE LibtorrentRasterbar::torrent-rasterbar)
elseif(TARGET LibtorrentRasterbar::torrent)
    target_link_libraries(${PROJECT_NAME} PRIVATE LibtorrentRasterbar::torrent)
elseif(TARGET LibtorrentRasterbar::torrent-rasterbar-static)
    target_link_libraries(${PROJECT_NAME} PRIVATE LibtorrentRasterbar::torrent-rasterbar-static)
else()
    target_link_libraries(${PROJECT_NAME} PRIVATE ${LIBTORRENT_LIBRARY})
endif()

# 为 clangd 设置包含路径
target_include_directories(${PROJECT_NAME} PRIVATE
    ${LIBTORRENT_INCLUDE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)

# 添加 Windows 系统依赖
if(MSVC)
    # 设置静态运行时库
    target_link_options(${PROJECT_NAME} PRIVATE
        /SUBSYSTEM:CONSOLE
    )

    # 添加必要的系统库
    target_link_libraries(${PROJECT_NAME} PRIVATE
        ws2_32
        iphlpapi
        crypt32
        bcrypt
    )
endif()

# 添加静态链接的编译器定义
target_compile_definitions(${PROJECT_NAME} PRIVATE
    TORRENT_USE_OPENSSL
    BOOST_ASIO_HAS_STD_CHRONO
    BOOST_ASIO_DISABLE_BOOST_REGEX
    _WIN32_WINNT=0x0601
)

# 禁用自动复制DLL
if(WIN32)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /MANIFEST:NO")
    set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} /MANIFEST:NO")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /MANIFEST:NO")
endif()

# 安装目标
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION bin
    BUNDLE DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib/static
)
