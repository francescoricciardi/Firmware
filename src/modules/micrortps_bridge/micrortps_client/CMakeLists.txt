############################################################################
#
#   Copyright (c) 2015 PX4 Development Team. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in
#    the documentation and/or other materials provided with the
#    distribution.
# 3. Neither the name PX4 nor the names of its contributors may be
#    used to endorse or promote products derived from this software
#    without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
# FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
# COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
# INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
# OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
# AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#
############################################################################

if(GENERATE_RTPS_BRIDGE)

	find_program(FASTRTPSGEN fastrtpsgen PATHS $ENV{FASTRTPSGEN_DIR})
	if(NOT FASTRTPSGEN)
		message(STATUS "WARNING: Unable to find fastrtpsgen. Building PX4 without RTPS bridge support")

		return()
	endif()

	# Do not delete everything in the current dir
	set(msg_out_path ${CMAKE_CURRENT_BINARY_DIR})

	if (NOT "${config_rtps_send_topics}" STREQUAL "" OR NOT "${config_rtps_receive_topics}" STREQUAL "")

		set(send_topic_files)
		foreach(topic ${config_rtps_send_topics})
			list(APPEND send_topic_files ${PX4_SOURCE_DIR}/msg/${topic}.msg)
		endforeach()

		set(receive_topic_files)
		foreach(topic ${config_rtps_receive_topics})
			list(APPEND receive_topic_files ${PX4_SOURCE_DIR}/msg/${topic}.msg)
		endforeach()

		foreach(topic ${config_rtps_send_topics})
			list(APPEND topic_bridge_files_out ${msg_out_path}/micrortps_agent/${topic}_Publisher.cpp)
			list(APPEND topic_bridge_files_out ${msg_out_path}/micrortps_agent/${topic}_Publisher.h)
		endforeach()

		foreach(topic ${config_rtps_receive_topics})
			list(APPEND topic_bridge_files_out ${msg_out_path}/micrortps_agent/${topic}_Subscriber.cpp)
			list(APPEND topic_bridge_files_out ${msg_out_path}/micrortps_agent/${topic}_Subscriber.h)
		endforeach()

		set(send_topic_files_opt)
		if (NOT "${send_topic_files}" STREQUAL "")
			set(send_topic_opt "--send")
		endif()

		set(receive_topic_files_opt)
		if (NOT "${receive_topic_files}" STREQUAL "")
			set(receive_topic_opt "--receive")
		endif()

		list(APPEND topic_bridge_files_out
			${msg_out_path}/micrortps_client/microRTPS_client.cpp
			${msg_out_path}/micrortps_client/microRTPS_transport.cpp
			${msg_out_path}/micrortps_client/microRTPS_transport.h
			)

		add_custom_command(OUTPUT ${topic_bridge_files_out}
			COMMAND ${PYTHON_EXECUTABLE} ${PX4_SOURCE_DIR}/msg/tools/generate_microRTPS_bridge.py
				--fastrtpsgen-dir $ENV{FASTRTPSGEN_DIR}
				${send_topic_opt} ${send_topic_files}
				${receive_topic_opt} ${receive_topic_files}
				--topic-msg-dir ${PX4_SOURCE_DIR}/msg
				--agent-outdir ${CMAKE_CURRENT_BINARY_DIR}/micrortps_agent
				--client-outdir ${CMAKE_CURRENT_BINARY_DIR}/micrortps_client
				>micrortps_bridge.log 2>&1 || cat micrortps_bridge.log # quiet successful build output
			DEPENDS ${send_topic_files} ${receive_topic_files}
			COMMENT "Generating RTPS topic bridge"
		)
		add_custom_target(topic_bridge_files DEPENDS ${topic_bridge_files_out})

		include_directories(${CMAKE_CURRENT_SOURCE_DIR})
		include_directories(${CMAKE_CURRENT_BINARY_DIR}/micrortps_client)

		px4_add_module(
			MODULE modules__micrortps_bridge__micrortps_client
			MAIN micrortps_client
			STACK_MAIN 4096
			SRCS
				microRTPS_client_main.cpp
				${msg_out_path}/micrortps_client/microRTPS_client.cpp
				${msg_out_path}/micrortps_client/microRTPS_transport.cpp
			DEPENDS
				platforms__common
				topic_bridge_files
				uorb_headers_microcdr
		)
		target_link_libraries(modules__micrortps_bridge__micrortps_client PRIVATE lib__micro-CDR)
	endif()
endif()