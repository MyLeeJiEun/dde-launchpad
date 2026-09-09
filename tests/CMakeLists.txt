# SPDX-FileCopyrightText: 2024 - 2026 UnionTech Software Technology Co., Ltd.
#
# SPDX-License-Identifier: CC0-1.0

find_package(Qt6 REQUIRED COMPONENTS Core Test)
find_package(Qt6 REQUIRED COMPONENTS Gui Qml Svg)  # needed by tested-source OBJECT libs
find_package(Qt6Core CONFIG REQUIRED Private)      # sortproxymodel.cpp uses QAbstractItemModelPrivate

# Coverage tools (required by the run-test target)
find_program(LCOV_BIN lcov)
find_program(GENHTML_BIN genhtml)

qt_standard_project_setup()

# ---------------------------------------------------------------------------
# Tested-source OBJECT libraries
#
# Per the long-term coverage rule: the tested source files (src/models/*.cpp,
# src/utils/*.cpp) are compiled as OBJECT libraries and linked *directly*
# into every test executable, instead of being reached only through the
# launchpadcommon.so shared library.  This ensures gcov .gcda runtime data
# is written to the test build directory and can be captured even in
# cross-prefix environments where the shared-library build path is not
# writable at runtime.
#
# launchpadcommon is still linked afterwards for transitive dependencies
# (D-Bus adaptors, dde-integration AppMgr, etc.).  Symbols defined by the
# OBJECT libs take precedence over the same symbols in the shared library,
# so the gcov-instrumented copies in the test executable are the ones that
# actually execute.
# ---------------------------------------------------------------------------

add_library(launchpad-test-models OBJECT
    ${CMAKE_SOURCE_DIR}/src/models/appsmodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/categorizedsortproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/countlimitproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/favoritedproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/freesortproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/frequentlyusedproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/itemarrangementproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/itemspage.cpp
    ${CMAKE_SOURCE_DIR}/src/models/itemspagemodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/multipagesortfilterproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/recentlyinstalledproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/searchfilterproxymodel.cpp
    ${CMAKE_SOURCE_DIR}/src/models/sortproxymodel.cpp
)
target_include_directories(launchpad-test-models PRIVATE
    ${CMAKE_SOURCE_DIR}/src/models
    ${CMAKE_SOURCE_DIR}/src/utils
    ${CMAKE_SOURCE_DIR}/src/ddeintegration
)
target_link_libraries(launchpad-test-models PUBLIC
    Qt6::Core
    Qt6::Gui
    Qt6::Qml        # QML_NAMED_ELEMENT / QML_SINGLETON in model headers (moc-generated registration)
    Qt6::CorePrivate # sortproxymodel.cpp -> QAbstractItemModelPrivate
    ${DTK_NS}::Core  # DConfig, DFileWatcherManager, DPinyin (appsmodel.cpp, frequentlyusedproxymodel.cpp)
)
# Coverage instrumentation for tested source
target_compile_options(launchpad-test-models PRIVATE --coverage)

add_library(launchpad-test-utils OBJECT
    ${CMAKE_SOURCE_DIR}/src/utils/blurhash.cpp
    ${CMAKE_SOURCE_DIR}/src/utils/categoryutils.cpp
    ${CMAKE_SOURCE_DIR}/src/utils/iconutils.cpp
)
target_include_directories(launchpad-test-utils PRIVATE
    ${CMAKE_SOURCE_DIR}/src/utils
)
target_link_libraries(launchpad-test-utils PUBLIC
    Qt6::Core
    Qt6::Gui
    Qt6::Svg
    ${DTK_NS}::Gui   # DIcon, DIconTheme, DSvgRenderer (iconutils.cpp)
)
# Coverage instrumentation for tested source
target_compile_options(launchpad-test-utils PRIVATE --coverage)

macro(launchpad_add_tests)
    foreach(_testname ${ARGN})
        qt_add_executable(launchpad-${_testname} ${_testname}.cpp)
        # OBJECT libs first: their symbols take precedence over launchpadcommon.so,
        # so the gcov-instrumented copies are the ones that execute at test time.
        target_link_libraries(launchpad-${_testname} PRIVATE
            Qt6::Core
            Qt6::Test
            launchpad-test-models
            launchpad-test-utils
            gio-utils          # already OBJECT; covers src/gioutils/ (e.g. gioappinfotest)
            launchpadcommon
        )
        # Coverage instrumentation for test executables
        target_compile_options(launchpad-${_testname} PRIVATE --coverage)
        target_link_options(launchpad-${_testname} PRIVATE --coverage)
        add_test(NAME launchpad-${_testname} COMMAND launchpad-${_testname})
        # Ensure `cmake --build build --target test` compiles all test executables
        # before ctest runs them.
        add_dependencies(test launchpad-${_testname})
    endforeach()
endmacro()

launchpad_add_tests(
    itemspagetest
    gioappinfotest
    searchfilterproxymodeltest
    countlimitproxymodeltest
    sortproxymodeltest
    freesortproxymodeltest
    recentlyinstalledproxymodeltest
    multipagesortfilterproxymodeltest
    categoryutilstest
    blurhashtest
    favoritedproxymodeltest
    frequentlyusedproxymodeltest
    itemarrangementproxymodeltest
    iconutilstest
    itemspagemodeltest
)

# ---------------------------------------------------------------------------
# run-test target: run all tests via ctest, then generate coverage HTML.
#
# Usage:
#   cmake --build build --target run-test
#
# Produces:
#   ${CMAKE_BINARY_DIR}/coverage_report/coverage.info          – raw lcov capture
#   ${CMAKE_BINARY_DIR}/coverage_report/coverage_filtered.info – filtered lcov data
#   ${CMAKE_BINARY_DIR}/coverage_report/html/                  – browsable HTML report
# ---------------------------------------------------------------------------

set(COVERAGE_DIR ${CMAKE_BINARY_DIR}/coverage_report)

if(LCOV_BIN AND GENHTML_BIN)
    add_custom_target(run-test
        # 1. Run all registered tests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        # 2. Prepare output directory
        COMMAND ${CMAKE_COMMAND} -E make_directory ${COVERAGE_DIR}
        # 3. Capture raw coverage data from gcov .gcda files
        COMMAND ${LCOV_BIN} --capture
                --directory ${CMAKE_BINARY_DIR}
                --output-file ${COVERAGE_DIR}/coverage.info
                --no-external
        # 4. Filter generated / non-project code
        COMMAND ${LCOV_BIN} --remove ${COVERAGE_DIR}/coverage.info
                '*/moc_*.cpp'
                '*/moc_*.h'
                '*/qrc_*.cpp'
                '*/ui_*.h'
                '*/qml_*.h'
                '*/qmlcache_*.cpp'
                '*/_autogen/*'
                '*_json.h'
                --output-file ${COVERAGE_DIR}/coverage_filtered.info
        # 5. Generate browsable HTML report
        COMMAND ${GENHTML_BIN} ${COVERAGE_DIR}/coverage_filtered.info
                --output-directory ${COVERAGE_DIR}/html
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        DEPENDS
            launchpad-itemspagetest
            launchpad-gioappinfotest
            launchpad-searchfilterproxymodeltest
            launchpad-countlimitproxymodeltest
            launchpad-sortproxymodeltest
            launchpad-freesortproxymodeltest
            launchpad-recentlyinstalledproxymodeltest
            launchpad-multipagesortfilterproxymodeltest
            launchpad-categoryutilstest
            launchpad-blurhashtest
            launchpad-favoritedproxymodeltest
            launchpad-frequentlyusedproxymodeltest
            launchpad-itemarrangementproxymodeltest
            launchpad-iconutilstest
            launchpad-itemspagemodeltest
        COMMENT "Running tests and generating coverage report in ${COVERAGE_DIR}"
    )
else()
    # Fallback when lcov/genhtml are not installed: run ctest only
    add_custom_target(run-test
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        DEPENDS
            launchpad-itemspagetest
            launchpad-gioappinfotest
            launchpad-searchfilterproxymodeltest
            launchpad-countlimitproxymodeltest
            launchpad-sortproxymodeltest
            launchpad-freesortproxymodeltest
            launchpad-recentlyinstalledproxymodeltest
            launchpad-multipagesortfilterproxymodeltest
            launchpad-categoryutilstest
            launchpad-blurhashtest
            launchpad-favoritedproxymodeltest
            launchpad-frequentlyusedproxymodeltest
            launchpad-itemarrangementproxymodeltest
            launchpad-iconutilstest
            launchpad-itemspagemodeltest
        COMMENT "Running tests (lcov/genhtml not found, skipping coverage report)"
    )
    if(NOT LCOV_BIN)
        message(WARNING "lcov not found; run-test will skip coverage report generation. Install lcov for full functionality.")
    endif()
    if(NOT GENHTML_BIN)
        message(WARNING "genhtml not found; run-test will skip coverage report generation. Install genhtml for full functionality.")
    endif()
endif()
