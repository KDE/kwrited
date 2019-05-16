cmake_minimum_required(VERSION 3.0)

project(kwrited)
set(PROJECT_VERSION "5.16.80")

set(QT_MIN_VERSION "5.12.0")
set(KF5_MIN_VERSION "5.58.0")

find_package(ECM ${KF5_MIN_VERSION} REQUIRED NO_MODULE)
set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH} ${ECM_KDE_MODULE_DIR})

include(CMakeDependentOption)
include(KDEInstallDirs)
include(KDECMakeSettings)
include(KDEFrameworkCompilerSettings NO_POLICY_SCOPE)
include(FeatureSummary)

find_package(Qt5 ${QT_MIN_VERSION} CONFIG REQUIRED COMPONENTS
    Widgets
)

find_package(KF5 ${KF5_MIN_VERSION} REQUIRED COMPONENTS
    CoreAddons
    I18n
    Pty
    Notifications
)

cmake_dependent_option(BUILD_AS_EXECUTABLE "Whether to build kwrited as executable" OFF
                       "KPty_HAVE_UTEMPTER" ON)

if (NOT BUILD_AS_EXECUTABLE)
find_package(KF5DBusAddons ${KF5_MIN_VERSION})
set_package_properties(KF5DBusAddons PROPERTIES DESCRIPTION "Convenience classes for DBus"
                       TYPE REQUIRED
                       PURPOSE "Needed to build kwrited's KDED module"
                      )
endif ()

add_definitions(-DQT_NO_NARROWING_CONVERSIONS_IN_CONNECT)
#add_definitions(-DQT_DISABLE_DEPRECATED_BEFORE=0x050a00)
add_definitions(-DQT_NO_URL_CAST_FROM_STRING)

# If libutempter is not available on the system, we build
# kwrited as an executable that is installed with setuid root
# (or setgid utmp) in order to be able to login successfully
# (and thus be useful, as kwrited does not work at all if it
# cannot login to the terminal device that it opens).

configure_file(config-kwrited.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config-kwrited.h)

if (BUILD_AS_EXECUTABLE)
    add_executable(kwrited kwrited.cpp)
    target_compile_definitions(kwrited PRIVATE -DPROJECT_VERSION="${PROJECT_VERSION}")
    target_link_libraries(kwrited  Qt5::Widgets KF5::CoreAddons KF5::I18n KF5::Notifications KF5::Pty)
    install(TARGETS kwrited  ${INSTALL_TARGETS_DEFAULT_ARGS} )
    if(IS_ABSOLUTE BIN_INSTALL_DIR)
        set(ABS_BIN_INSTALL_DIR "${BIN_INSTALL_DIR}")
    else()
        set(ABS_BIN_INSTALL_DIR "\${CMAKE_INSTALL_PREFIX}/${BIN_INSTALL_DIR}")
    endif()
    install(CODE "
               set(KWRITED_PATH \"\$ENV{DESTDIR}${ABS_BIN_INSTALL_DIR}/kwrited\")

               # On Debian we setgid instead of setuid.
               # Feel free to add checks here for other distros as well, if needed.
               if (EXISTS /etc/debian_version)
                   set(SETGID 1)
                   set(SETGID_GROUP utmp)
               endif ()

               if (SETGID)
                   execute_process(COMMAND sh -c \"chgrp \${SETGID_GROUP} '\${KWRITED_PATH}' && chmod g+s '\${KWRITED_PATH}'\")
               else ()
                   execute_process(COMMAND sh -c \"chown root '\${KWRITED_PATH}' && chmod u+s '\${KWRITED_PATH}'\")
               endif()
           ")

    install(FILES kwrited-autostart.desktop DESTINATION ${AUTOSTART_INSTALL_DIR})
else ()
    kcoreaddons_add_plugin(kded_kwrited
        SOURCES kwrited.cpp
        JSON kwrited.json
        INSTALL_NAMESPACE "kf5/kded"
    )
    set_target_properties(kded_kwrited PROPERTIES OUTPUT_NAME kwrited)
    target_compile_definitions(kded_kwrited PRIVATE -DPROJECT_VERSION="${PROJECT_VERSION}")
    target_link_libraries(kded_kwrited KF5::CoreAddons KF5::DBusAddons KF5::I18n KF5::Notifications KF5::Pty)
endif ()

# The .notifyrc file is common
install(FILES kwrited.notifyrc DESTINATION ${KNOTIFYRC_INSTALL_DIR})

feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES FATAL_ON_MISSING_REQUIRED_PACKAGES)
