project(gmic-qt)

message(STATUS "Using CMake version: ${CMAKE_VERSION}")

cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)
LIST (APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules")
include(FeatureSummary)

set(CMAKE_CXX_STANDARD 11)
add_definitions(-Dcimg_use_cpp11=1)
set(MIN_QT_VERSION 5.2.0)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC OFF)
set(CMAKE_AUTORCC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(EXTRA_LIBRARIES)

set (GMIC_QT_HOST "gimp" CACHE STRING "Define for which host qmic-qt will be built: gimp, krita, none, paintdotnet, or digikam.")
if (${GMIC_QT_HOST} STREQUAL "none")
  message("Building standalone version.")
else()
  message("Building for target host application: " ${GMIC_QT_HOST})
endif()

if(EXISTS "../src/gmic.cpp")
  set (GMIC_PATH "../src" CACHE STRING "Define the path to the gmic headers")
else()
  set (GMIC_PATH "../gmic/src" CACHE STRING "Define the path to the gmic headers")
endif()

message("G'MIC path: " ${GMIC_PATH})

option(ENABLE_DYNAMIC_LINKING "Dynamically link the binaries to the GMIC shared library" OFF)
option(ENABLE_CURL "Add support for curl" ON)
set (GMIC_LIB_PATH "${GMIC_PATH}" CACHE STRING "Define the path to the GMIC shared library")

option(ENABLE_ASAN "Enable -fsanitize=address (if debug build)" ON)
option(ENABLE_FFTW3 "Enable FFTW3 library support" ON)

option(ENABLE_LTO "Enable -flto (Link Time Optimizer) on gcc and clang" ON)
if (WIN32 AND ENABLE_LTO)
  message(WARNING "LTO is disabled on windows")
  set(ENABLE_LTO OFF)
endif()

#
# Look for G'MIC repository
#
get_filename_component(GMIC_ABSOLUTE_PATH ${GMIC_PATH} ABSOLUTE BASEDIR ${CMAKE_SOURCE_DIR})
if (EXISTS ${GMIC_ABSOLUTE_PATH}/gmic.cpp)
  message("Found G'MIC repository")
else()
  get_filename_component(TARGET_CLONE_DIR ${GMIC_ABSOLUTE_PATH}/.. ABSOLUTE)
  message("")
  message("Cannot find G'MIC repository in " ${GMIC_ABSOLUTE_PATH} )
  message("")
  message("You should try:")
  message("")
  message("   git clone https://github.com/dtschump/gmic.git " ${TARGET_CLONE_DIR}/gmic )
  message("")
  message(FATAL_ERROR "\nG'MIC repository not found")
endif()

#
# Look for CImg.h and gmic_stdlib.h
#
set(GMIC_FILES CImg.h gmic_stdlib.h)
foreach(F ${GMIC_FILES})
  if(EXISTS ${GMIC_ABSOLUTE_PATH}/${F})
    message("Found " ${GMIC_PATH}/${F})
  else()
    message(${F} " not found")
    execute_process(COMMAND make -C ${GMIC_ABSOLUTE_PATH} ${F})
    if(EXISTS ${GMIC_ABSOLUTE_PATH}/${F})
      message("Found " ${GMIC_PATH}/${F})
    else()
      message(FATAL_ERROR "\nCannot obtain " ${GMIC_PATH}/${F})
    endif()
  endif()
endforeach()

#
# Ensure that gmic and CImg are the same version
#
file(STRINGS ${GMIC_ABSOLUTE_PATH}/CImg.h CIMG_VERSION REGEX "cimg_version ")
string(REGEX REPLACE ".*cimg_version " "" CIMG_VERSION ${CIMG_VERSION})
message("CImg version is [" ${CIMG_VERSION} "]")

file(STRINGS ${GMIC_ABSOLUTE_PATH}/gmic.h GMIC_VERSION REGEX "gmic_version ")
string(REGEX REPLACE ".*gmic_version " "" GMIC_VERSION ${GMIC_VERSION})
message("G'MIC version is [" ${GMIC_VERSION} "]")

if (NOT(${GMIC_VERSION} EQUAL ${CIMG_VERSION}))
  message(FATAL_ERROR "\nVersion numbers of files 'gmic.h' (" ${GMIC_VERSION} ") and 'CImg.h' (" ${CIMG_VERSION} ") mismatch")
endif()


option(PRERELEASE "Set to ON makes this a prelease build")
if (${PRERELEASE})
    string(TIMESTAMP PRERELEASE_DATE %y%m%d)
    message("Prelease date is " ${PRERELEASE_DATE})
    add_definitions(-Dgmic_prerelease="${PRERELEASE_DATE}")
endif()

option(DRMINGW "Set to ON enables the drmingw debugger.")
if (${DRMINGW})
    add_definitions(-DDRMINGW)
endif()



# Required packages

#
# Qt5
#
find_package(Qt5 ${MIN_QT_VERSION}
        REQUIRED COMPONENTS
        Core
        Gui
        Widgets
        Network
)

#
# For the translations
#
find_package(Qt5LinguistTools REQUIRED)

#
# PNG
#
find_package(PNG REQUIRED)
add_definitions(${PNG_DEFINITIONS})
add_definitions(-Dcimg_use_png)
include_directories(SYSTEM ${PNG_INCLUDE_DIR})
if (APPLE)
    # this is not added correctly on OSX -- see http://forum.kde.org/viewtopic.php?f=139&t=101867&p=221242#p221242
    include_directories(SYSTEM ${PNG_INCLUDE_DIR})
endif()

#
# ZLIB
#
find_package(ZLIB REQUIRED)
add_definitions(-Dcimg_use_zlib)
include_directories(SYSTEM ${ZLIB_INCLUDE_DIRS} )

#
# FFTW3
#
find_package(FFTW3 REQUIRED)
add_definitions(-Dcimg_use_fftw3 )
include_directories(${FFTW3_INCLUDE_DIR})
find_library(FFTW3_THREADS_LIB fftw3_threads PATHS ${FFTW3_LIBRARY_DIRS})
if(FFTW3_THREADS_LIB STREQUAL "FFTW3_THREADS_LIB-NOTFOUND")
    add_definitions(-Dcimg_use_fftw3_singlethread)
else()
    list(APPEND EXTRA_LIBRARIES "-lfftw3_threads")
endif()

#
# CURL
#
if(ENABLE_CURL)
    find_package(CURL)
    if (CURL_FOUND)
        add_definitions(-Dcimg_use_curl)
        include_directories(SYSTEM ${CURL_INCLUDE_DIRS} )
    endif()
endif()

#
# Test for OpenMP
#
find_package(OpenMP)
set_package_properties(OpenMP PROPERTIES
    DESCRIPTION "A low-level parallel execution library"
    URL "http://openmp.org/wp/"
    TYPE OPTIONAL
    PURPOSE "Optionally used by gmic-qt")

if (CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER 4.8.3 AND OPENMP_FOUND)
    message("G'Mic: using OpenMP")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    add_definitions(-Dcimg_use_openmp)
    add_definitions(-fopenmp)
endif()

#
# LTO option
#

if (ENABLE_LTO AND (CMAKE_COMPILER_IS_GNUCC OR (CMAKE_CSS_COMPILER_IS STREQUAL "Clang")))
    message("Link Time Optimizer enabled")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -flto")
endif()

#
# add all defines
#

set(gmic_qt_LIBRARIES
        Qt5::Core
        Qt5::Widgets
        Qt5::Gui
        Qt5::Network
        ${PNG_LIBRARIES}
        ${FFTW3_LIBRARIES}
        ${ZLIB_LIBRARIES}
        ${EXTRA_LIBRARIES}
)

if(ENABLE_CURL)
    if (CURL_FOUND)
        set(gmic_qt_LIBRARIES
            ${gmic_qt_LIBRARIES}
            ${CURL_LIBRARIES}
        )
    endif()
endif()

add_definitions(-Dgmic_build)
add_definitions(-Dcimg_use_abort)
add_definitions(-Dgmic_is_parallel)
add_definitions(-Dgmic_gui)
add_definitions(-Dcimg_use_abort)
add_definitions(-Dcimg_appname=\"gmic\")

if (UNIX AND NOT APPLE)
    add_definitions(-Dcimg_display=1)
    add_definitions(-D_IS_LINUX_)
    add_definitions(-Dcimg_use_vt100)
    add_definitions(-D_IS_UNIX_)
    find_package(X11)
    set(gmic_qt_LIBRARIES
        ${gmic_qt_LIBRARIES}
        ${X11_LIBRARIES} # XXX: Search for X11: Wayland is coming!
    )

endif()

if (APPLE)
    add_definitions(-Dcimg_display=0)
    add_definitions(-D_IS_MACOS_)
    set(CMAKE_MACOSX_RPATH 1)
    set(BUILD_WITH_INSTALL_RPATH 1)
    add_definitions(-mmacosx-version-min=10.9 -Wno-macro-redefined -Wno-deprecated-register)
endif()

if (WIN32)
    add_definitions(-Dcimg_display=2)
    add_definitions(-DPSAPI_VERSION=1)
    set(gmic_qt_LIBRARIES
        ${gmic_qt_LIBRARIES}
        pthread psapi gdi32
    )
endif()

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
SET(CMAKE_INSTALL_RPATH "$ORIGIN/")

if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    message("Debug build")
    add_definitions(-D_GMIC_QT_DEBUG_)
    if(ENABLE_ASAN)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
    endif(ENABLE_ASAN)
elseif (CMAKE_BUILD_TYPE STREQUAL "Release")
    message("Release build")
    add_definitions(-DQT_NO_DEBUG_OUTPUT)
    string(REPLACE "-O2" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
    string(REPLACE "-O3" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -s")
    if (WIN32)
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -mwindows")
    endif()
elseif (CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    message("Release build with debug info")
    add_definitions(-DQT_NO_DEBUG_OUTPUT)
    string(REPLACE "-O2" "" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
    string(REPLACE "-O3" "" CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
    set_source_files_properties(${GMIC_PATH}/gmic.cpp PROPERTIES COMPILE_FLAGS "-Ofast")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -O2")
else()
    message(FATAL_ERROR "Build type not recognized (${CMAKE_BUILD_TYPE})")
endif()

include_directories(${CMAKE_SOURCE_DIR}/src ${GMIC_PATH})

set (gmic_qt_SRCS

  src/ClickableLabel.h
  src/Common.h
  src/OverrideCursor.h
  src/DialogSettings.h
  src/FilterParameters/AbstractParameter.h
  src/FilterParameters/BoolParameter.h
  src/FilterParameters/ButtonParameter.h
  src/FilterParameters/ChoiceParameter.h
  src/FilterParameters/ColorParameter.h
  src/FilterParameters/ConstParameter.h
  src/FilterParameters/CustomDoubleSpinBox.h
  src/FilterParameters/FileParameter.h
  src/FilterParameters/FilterParametersWidget.h
  src/FilterParameters/FloatParameter.h
  src/FilterParameters/FolderParameter.h
  src/FilterParameters/IntParameter.h
  src/FilterParameters/LinkParameter.h
  src/FilterParameters/MultilineTextParameterWidget.h
  src/FilterParameters/NoteParameter.h
  src/FilterParameters/PointParameter.h
  src/FilterParameters/SeparatorParameter.h
  src/FilterParameters/TextParameter.h
  src/FilterSelector/FiltersModel.h
  src/FilterSelector/FiltersModelReader.h
  src/FilterSelector/FiltersPresenter.h
  src/FilterSelector/FiltersView/FiltersView.h
  src/FilterSelector/FiltersView/TreeView.h
  src/FilterSelector/FiltersVisibilityMap.h
  src/CroppedImageListProxy.h
  src/CroppedActiveLayerProxy.h
  src/FilterSyncRunner.h
  src/FilterThread.h
  src/Globals.h
  src/gmic_qt.h
  src/GmicStdlib.h
  src/GmicProcessor.h
  src/HeadlessProcessor.h
  src/Host/host.h
  src/HtmlTranslator.h
  src/IconLoader.h
  src/ImageConverter.h
  src/ImageTools.h
  src/InputOutputState.h
  src/KeypointList.h
  src/LayersExtentProxy.h
  src/Logger.h
  src/MainWindow.h
  src/ParametersCache.h
  src/PreviewMode.h
  src/TimeLogger.h
  src/Updater.h
  src/Utils.h
  src/FilterSelector/FiltersView/FilterTreeFolder.h
  src/FilterSelector/FiltersView/FilterTreeItem.h
  src/FilterSelector/FavesModel.h
  src/FilterSelector/FavesModelReader.h
  src/FilterSelector/FiltersView/FilterTreeAbstractItem.h
  src/FilterSelector/FiltersView/FilterTreeItemDelegate.h
  src/FilterSelector/FavesModelWriter.h
  src/Widgets/ProgressInfoWidget.h
  src/Widgets/PreviewWidget.h
  src/Widgets/InOutPanel.h
  src/Widgets/ZoomLevelSelector.h
  src/Widgets/SearchFieldWidget.h
  src/Widgets/LanguageSelectionWidget.h
  src/Widgets/ProgressInfoWindow.h
  src/ZoomConstraint.h

  ${GMIC_PATH}/gmic.h
  ${GMIC_PATH}/CImg.h
  ${GMIC_PATH}/gmic_stdlib.h

  src/ClickableLabel.cpp
  src/Common.cpp
  src/OverrideCursor.cpp
  src/DialogSettings.cpp
  src/FilterParameters/AbstractParameter.cpp
  src/FilterParameters/BoolParameter.cpp
  src/FilterParameters/ButtonParameter.cpp
  src/FilterParameters/ChoiceParameter.cpp
  src/FilterParameters/ColorParameter.cpp
  src/FilterParameters/ConstParameter.cpp
  src/FilterParameters/CustomDoubleSpinBox.cpp
  src/FilterParameters/FileParameter.cpp
  src/FilterParameters/FilterParametersWidget.cpp
  src/FilterParameters/FloatParameter.cpp
  src/FilterParameters/FolderParameter.cpp
  src/FilterParameters/IntParameter.cpp
  src/FilterParameters/LinkParameter.cpp
  src/FilterParameters/MultilineTextParameterWidget.cpp
  src/FilterParameters/NoteParameter.cpp
  src/FilterParameters/PointParameter.cpp
  src/FilterParameters/SeparatorParameter.cpp
  src/FilterParameters/TextParameter.cpp
  src/FilterSelector/FiltersModel.cpp
  src/FilterSelector/FiltersModelReader.cpp
  src/FilterSelector/FiltersPresenter.cpp
  src/FilterSelector/FiltersView/FiltersView.cpp
  src/FilterSelector/FiltersView/TreeView.cpp
  src/FilterSelector/FiltersVisibilityMap.cpp
  src/CroppedImageListProxy.cpp
  src/CroppedActiveLayerProxy.cpp
  src/FilterSyncRunner.cpp
  src/FilterThread.cpp
  src/gmic_qt.cpp
  src/Globals.cpp
  src/GmicStdlib.cpp
  src/GmicProcessor.cpp
  src/HeadlessProcessor.cpp
  src/HtmlTranslator.cpp
  src/IconLoader.cpp
  src/ImageConverter.cpp
  src/ImageTools.cpp
  src/InputOutputState.cpp
  src/KeypointList.cpp
  src/LayersExtentProxy.cpp
  src/Logger.cpp
  src/MainWindow.cpp
  src/ParametersCache.cpp
  src/PreviewMode.cpp
  src/TimeLogger.cpp
  src/Updater.cpp
  src/Utils.cpp
  src/FilterSelector/FiltersView/FilterTreeItem.cpp
  src/FilterSelector/FiltersView/FilterTreeFolder.cpp
  src/FilterSelector/FavesModel.cpp
  src/FilterSelector/FavesModelReader.cpp
  src/FilterSelector/FiltersView/FilterTreeAbstractItem.cpp
  src/FilterSelector/FiltersView/FilterTreeItemDelegate.cpp
  src/FilterSelector/FavesModelWriter.cpp
  src/Widgets/PreviewWidget.cpp
  src/Widgets/ProgressInfoWidget.cpp
  src/Widgets/InOutPanel.cpp
  src/Widgets/ZoomLevelSelector.cpp
  src/Widgets/SearchFieldWidget.cpp
  src/Widgets/LanguageSelectionWidget.cpp
  src/Widgets/ProgressInfoWindow.cpp
)

if(ENABLE_DYNAMIC_LINKING)
  set(CMAKE_SKIP_RPATH TRUE)
  set(gmic_qt_LIBRARIES
    ${gmic_qt_LIBRARIES}
    "gmic"
    )
  link_directories(${GMIC_LIB_PATH})
else(ENABLE_DYNAMIC_LINKING)
  set(gmic_qt_SRCS
    ${gmic_qt_SRCS}
    ${GMIC_PATH}/gmic.cpp
    )
endif(ENABLE_DYNAMIC_LINKING)

qt5_wrap_ui(gmic_qt_SRCS
  ui/dialogsettings.ui
  ui/filtersview.ui
  ui/headlessprogressdialog.ui
  ui/inoutpanel.ui
  ui/languageselectionwidget.ui
  ui/mainwindow.ui
  ui/multilinetextparameterwidget.ui
  ui/progressinfowidget.ui
  ui/progressinfowindow.ui
  ui/SearchFieldWidget.ui
  ui/zoomlevelselector.ui
)

set(gmic_translation_files

    translations/cs.ts
    translations/de.ts
    translations/es.ts
    translations/fr.ts
    translations/id.ts
    translations/it.ts
    translations/nl.ts
    translations/pl.ts
    translations/pt.ts
    translations/ru.ts
    translations/ua.ts
    translations/ja.ts
    translations/zh.ts
)

set_source_files_properties(${gmic_translation_files} PROPERTIES OUTPUT_LOCATION translations)

# qt5_create_translation(
#     qmic_qt_QM
#     ${CMAKE_SOURCE_DIR}/translations
#     ${gmic_qt_SRCS}
#     ${gmic_translation_files}
# )

qt5_add_translation(gmic_qt_QM
    ${gmic_translation_files}
  )

# install(FILES ${gmic_qt_QM} DESTINATION ${CMAKE_SOURCE_DIR}/translations)

set(gmic_qt_QRC
    gmic_qt.qrc
    translations.qrc
)

if (${GMIC_QT_HOST} STREQUAL "gimp")

    execute_process(COMMAND gimptool-2.0 --libs-noui OUTPUT_VARIABLE GIMP2_LIBRARIES OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND gimptool-2.0 --cflags-noui OUTPUT_VARIABLE GIMP2_INCLUDE_DIRS OUTPUT_STRIP_TRAILING_WHITESPACE)
    execute_process(COMMAND pkg-config gimp-2.0 --define-variable=prefix=${CMAKE_INSTALL_PREFIX} --variable gimplibdir OUTPUT_VARIABLE GIMP2_PKGLIBDIR OUTPUT_STRIP_TRAILING_WHITESPACE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GIMP2_INCLUDE_DIRS}")

    set (gmic_qt_SRCS ${gmic_qt_SRCS} src/Host/Gimp/host_gimp.cpp)
    add_definitions(-DGMIC_HOST=gimp_qt -DGIMP_DISABLE_DEPRECATED)
    add_executable(gmic_gimp_qt ${gmic_qt_SRCS} ${gmic_qt_QRC} ${qmic_qt_QM})
    target_link_libraries(
      gmic_gimp_qt
      PRIVATE
      ${GIMP2_LIBRARIES}
      ${gmic_qt_LIBRARIES}
      )
    install(TARGETS gmic_gimp_qt RUNTIME DESTINATION "${GIMP2_PKGLIBDIR}/plug-ins")

elseif (${GMIC_QT_HOST} STREQUAL "krita")

    set (gmic_qt_SRCS ${gmic_qt_SRCS} src/Host/Krita/host_krita.cpp)
    add_definitions(-DGMIC_HOST=krita)
    add_executable(gmic_krita_qt ${gmic_qt_SRCS} ${gmic_qt_QRC} ${qmic_qt_QM})
    target_link_libraries(
      gmic_krita_qt
      PRIVATE
      ${gmic_qt_LIBRARIES}
      )
    install(TARGETS gmic_krita_qt RUNTIME DESTINATION bin)

elseif (${GMIC_QT_HOST} STREQUAL "none")

    set (gmic_qt_SRCS ${gmic_qt_SRCS} src/Host/None/host_none.cpp src/Host/None/ImageDialog.h src/Host/None/ImageDialog.cpp)
    add_definitions(-DGMIC_HOST=standalone)
    add_executable(gmic_qt ${gmic_qt_SRCS} ${gmic_qt_QRC}  ${qmic_qt_QM})
    target_link_libraries(gmic_qt PRIVATE ${gmic_qt_LIBRARIES})
    install(TARGETS gmic_qt RUNTIME DESTINATION bin)

elseif (${GMIC_QT_HOST} STREQUAL "paintdotnet")

    set (gmic_qt_SRCS ${gmic_qt_SRCS} src/Host/PaintDotNet/host_paintdotnet.cpp)
    add_definitions(-DGMIC_HOST=paintdotnet)
    add_executable(gmic_paintdotnet_qt ${gmic_qt_SRCS} ${gmic_qt_QRC} ${qmic_qt_QM})
    target_link_libraries(
      gmic_paintdotnet_qt
      PRIVATE
      ${gmic_qt_LIBRARIES}
      )

elseif (${GMIC_QT_HOST} STREQUAL "digikam")

    include(GNUInstallDirs)

    find_package(DigikamCore "6.3.0" CONFIG REQUIRED)

    set_package_properties(DigikamCore PROPERTIES
                           URL "http://www.digikam.org"
                           DESCRIPTION "digiKam core library"
    )

    # --- Manage C++ exception rules -------

    string(REPLACE " -DQT_NO_EXCEPTIONS " " " CMAKE_CXX_FLAGS " ${CMAKE_CXX_FLAGS} ")
    string(REPLACE " -fno-exceptions "    " " CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    string(STRIP "${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS)

    if (MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -EHsc")
    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
        if (WIN32)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -EHsc")
        else()
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
        endif()
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
    endif()

    string(STRIP "${CMAKE_CXX_FLAGS}" ${CMAKE_CXX_FLAGS})

    # ---

    include_directories($<TARGET_PROPERTY:Digikam::digikamcore,INTERFACE_INCLUDE_DIRECTORIES>/digikam)

    set (gmic_qt_SRCS ${gmic_qt_SRCS} src/Host/digiKam/host_digikam.cpp src/Host/digiKam/gmicqttoolplugin.cpp)
    add_definitions(-DGMIC_HOST=digikam)
    add_library(Editor_GmicQt_Plugin
                MODULE ${gmic_qt_SRCS} ${gmic_qt_QRC} ${qmic_qt_QM})

    set_target_properties(Editor_GmicQt_Plugin PROPERTIES PREFIX "")

    target_link_libraries(Editor_GmicQt_Plugin
                          PRIVATE
                          ${gmic_qt_LIBRARIES}
                          Digikam::digikamcore)

    # --- Install rules ---------------------------------

    get_target_property(QT_QMAKE_EXECUTABLE ${Qt5Core_QMAKE_EXECUTABLE} IMPORTED_LOCATION)

    if(NOT QT_QMAKE_EXECUTABLE)
        message(FATAL_ERROR "qmake is not found.")
    endif()

    # execute the command "qmake -query QT_INSTALL_PLUGINS" to get the path of plugins dir.
    execute_process(COMMAND ${QT_QMAKE_EXECUTABLE} -query QT_INSTALL_PLUGINS
                    OUTPUT_VARIABLE QT_PLUGINS_DIR
                    OUTPUT_STRIP_TRAILING_WHITESPACE)

    if(NOT QT_PLUGINS_DIR)
        message(FATAL_ERROR "Qt5 plugin directory cannot be detected.")
    endif()

    install(TARGETS Editor_GmicQt_Plugin
            DESTINATION ${QT_PLUGINS_DIR}/digikam/editor)

else()
    message(FATAL_ERROR "GMIC_QT_HOST is not defined as gimp, krita, none, paintdotnet, or digikam")
endif()

feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)
