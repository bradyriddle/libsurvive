project(libsurvive)
cmake_minimum_required(VERSION 3.8.2)

enable_testing()
IF(UNIX)
	SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC -flto -std=gnu99 -rdynamic -Werror=incompatible-pointer-types -Wall -Wno-unused-variable -Wno-switch -Wno-parentheses -Wno-missing-braces")
ENDIF()

include_directories(redist include/libsurvive include)

option(USE_HIDAPI "Use HIDAPI instead of libusb" OFF)
option(USE_ASAN "Use address sanitizer" OFF)

if(USE_ASAN)
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -fsanitize=undefined")
endif()

if(USE_HIDAPI OR WIN32)
    add_definitions (-DHIDAPI)
	IF(WIN32)
		SET(SURVIVE_SRCS ${SURVIVE_SRCS} ./redist/hid-windows.c ./winbuild/getdelim.c)
	else()
		list(APPEND ADDITIONAL_LIBRARIES udev hidapi-libusb)
	endif()
endif()

IF(WIN32)
	add_definitions(-DNOZLIB)
ENDIF()

SET(SURVIVE_SRCS ${SURVIVE_SRCS}
        ./include/libsurvive/poser.h
		./include/libsurvive/survive.h
        ./include/libsurvive/survive_api.h
        ./include/libsurvive/survive_optimizer.h
        ./include/libsurvive/survive_reproject.h
        ./include/libsurvive/survive_types.h
        ./redist/crc32.c
        ./redist/glutil.c
        ./redist/jsmn.c
        ./redist/jsmntest.c
        ./redist/json_helpers.c
        ./redist/linmath.c
        ./redist/mpfit/mpfit.c
        ./redist/puff.c
		./redist/symbol_enumerator.c
        ./src/ootx_decoder.c
        ./src/poser.c
        ./src/poser_general_optimizer.c
        ./src/survive.c
        ./src/survive_api.c
        ./src/survive_cal.c
        ./src/survive_config.c
        ./src/survive_default_devices.c
        ./src/survive_disambiguator.c
        ./src/survive_driverman.c
        ./src/survive_imu.c
        ./src/survive_optimizer.c
        ./src/survive_playback.c        
		./src/survive_plugins.c
        ./src/survive_process.c
        ./src/survive_reproject.c
        ./src/survive_reproject.generated.h
        ./src/survive_sensor_activations.c
        ./src/survive_usb.c)

add_library(survive SHARED ${SURVIVE_SRCS})

add_subdirectory(redist)

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

target_link_libraries(survive minimal_opencv Threads::Threads ${CMAKE_DL_LIBS} ${ADDITIONAL_LIBRARIES})
IF(NOT WIN32)
	target_link_libraries(survive z m usb-1.0 )
else()
	target_link_libraries(survive DbgHelp SetupAPI)
endif()
#set_target_properties(survive PROPERTIES LIBRARY_OUTPUT_DIRECTORY "lib")
#set_target_properties(survive PROPERTIES RUNTIME_OUTPUT_DIRECTORY "lib")


foreach(executable calibrate data_recorder simple_pose_test survive-cli)
    add_executable(${executable} ${executable}.c )
    target_link_libraries(${executable} survive)
endforeach()
target_link_libraries(calibrate CNGFX)
target_link_libraries(simple_pose_test CNGFX)

SET(PLUGINS
        driver_dummy driver_vive
        disambiguator_turvey disambiguator_statebased disambiguator_charles
        poser_dummy poser_mpfit poser_epnp poser_sba poser_imu poser_charlesrefine
)

find_library(PCAP_LIBRARY pcap)
if(PCAP_LIBRARY)
    list(APPEND PLUGINS driver_usbmon)
    set(driver_usbmon_ADDITIONAL_LIBS "${PCAP_LIBRARY}")
else()
    message("Can't build usbmon plugin -- pcap library was not found")
endif()

if(UNIX)
	list(APPEND PLUGINS driver_udp)
endif()

set(poser_sba_ADDITIONAL_LIBS sba)
set(poser_epnp_ADDITIONAL_SRCS src/epnp/epnp.c)
foreach(PLUGIN ${PLUGINS})
    add_library(${PLUGIN} SHARED src/${PLUGIN}.c ${${PLUGIN}_ADDITIONAL_SRCS})
    target_link_libraries(${PLUGIN} survive ${${PLUGIN}_ADDITIONAL_LIBS})
    set_target_properties(${PLUGIN} PROPERTIES PREFIX "")
    set_target_properties(${PLUGIN} PROPERTIES LIBRARY_OUTPUT_DIRECTORY "plugins")
	set_target_properties(${PLUGIN} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_DEBUG "Debug/plugins")
	set_target_properties(${PLUGIN} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_RELEASE "Release/plugins")
endforeach()

add_subdirectory(src/test_cases)
