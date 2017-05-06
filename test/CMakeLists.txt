enable_testing()

file(GLOB files "${CMAKE_CURRENT_SOURCE_DIR}/*")
# work around for cmake restriction: test cases cannot depend on non executable targets.
# So we need to implement the test cases as make targets and build dummy test cases.
# This is slow, but can't help. The add_test command is very feature incomplete.
foreach(filename ${files})

	# Assembler tests with simple output
	string(REGEX REPLACE ".*gpu_fft_(.*)\\.qasm$" "\\1" name ${filename})
	if(NOT ${name} STREQUAL ${filename})
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/shader_${name}.strip
			COMMAND sed s/\\s*\\/\\/.*// ${CMAKE_CURRENT_SOURCE_DIR}/shader_${name}.hex > ${CMAKE_CURRENT_BINARY_DIR}/shader_${name}.strip
			DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/shader_${name}.hex VERBATIM )
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.passed ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.diff ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.hex
			COMMAND $<TARGET_FILE:vc4asm> -V -C ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.hex -i vc4.qinc ${filename}
			COMMAND diff ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.hex ${CMAKE_CURRENT_BINARY_DIR}/shader_${name}.strip > ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.diff
			COMMAND touch ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.passed # Work around for cmake bug #15474
			DEPENDS vc4asm ${filename} ${CMAKE_CURRENT_BINARY_DIR}/shader_${name}.strip VERBATIM )
		add_custom_target( test_qasm_${name} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/gpu_fft_${name}.passed )
		list(APPEND test_qasm test_qasm_${name} )
		add_test(NAME qasm_${name} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
			COMMAND ${CMAKE_COMMAND} --build . --target test_qasm_${name} --config $<CONFIGURATION> )
	endif()

	#Test of C output with header
	string(REGEX REPLACE ".*(rpi_.*)\\.qasm$" "\\1" name ${filename})
	if(NOT ${name} STREQUAL ${filename})
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed ${CMAKE_CURRENT_BINARY_DIR}/${name}.c ${CMAKE_CURRENT_BINARY_DIR}/${name}.h ${CMAKE_CURRENT_BINARY_DIR}/${name}.strip.c ${CMAKE_CURRENT_BINARY_DIR}/${name}.diff
			COMMAND $<TARGET_FILE:vc4asm> -v -c ${CMAKE_CURRENT_BINARY_DIR}/${name}.c -h ${CMAKE_CURRENT_BINARY_DIR}/${name}.h -i vc4.qinc ${filename}
			COMMAND perl -pe "s/\\/\\/\\s+(.*?)(?:\\s*#.*|$)/\\/\\/ \\1/" ${CMAKE_CURRENT_BINARY_DIR}/${name}.c > ${CMAKE_CURRENT_BINARY_DIR}/${name}.strip.c
			COMMAND diff ${CMAKE_CURRENT_BINARY_DIR}/${name}.strip.c ${CMAKE_CURRENT_SOURCE_DIR}/${name}.ref.c > ${CMAKE_CURRENT_BINARY_DIR}/${name}.diff
			COMMAND diff ${CMAKE_CURRENT_BINARY_DIR}/${name}.h ${CMAKE_CURRENT_SOURCE_DIR}/${name}.ref.h >> ${CMAKE_CURRENT_BINARY_DIR}/${name}.diff
			COMMAND touch ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed # Work around for cmake bug #15474
			DEPENDS vc4asm ${filename} ${CMAKE_CURRENT_SOURCE_DIR}/${name}.ref.c ${CMAKE_CURRENT_SOURCE_DIR}/${name}.ref.h VERBATIM )
		add_custom_target( test_cout_${name} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed )
		list(APPEND test_cout test_cout_${name} )
		add_test(NAME cout_${name} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
			COMMAND ${CMAKE_COMMAND} --build . --target test_cout_${name} --config $<CONFIGURATION> )
	endif()

	# Parser tests (errors and warnings)
	string(REGEX REPLACE ".*(parser\\..*)\\.qasm$" "\\1" name ${filename})
	if(NOT ${name} STREQUAL ${filename})
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed ${CMAKE_CURRENT_BINARY_DIR}/${name}.check ${CMAKE_CURRENT_BINARY_DIR}/${name}.out
			COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/ignoreResult.pl $<TARGET_FILE:vc4asm> -V -Q -i vc4.qinc ${filename} > ${CMAKE_CURRENT_BINARY_DIR}/${name}.out 2>&1
			COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/checkOutput.pl ${filename} < ${CMAKE_CURRENT_BINARY_DIR}/${name}.out > ${CMAKE_CURRENT_BINARY_DIR}/${name}.check
			COMMAND cat ${CMAKE_CURRENT_BINARY_DIR}/${name}.check 2> ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed # Work around for cmake bug #15474
			DEPENDS vc4asm ${filename} ${CMAKE_CURRENT_SOURCE_DIR}/checkOutput.pl ${CMAKE_CURRENT_SOURCE_DIR}/ignoreResult.pl VERBATIM )
		add_custom_target( test_${name} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed )
		list(APPEND test_parser test_${name} )
		add_test(NAME ${name} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
			COMMAND ${CMAKE_COMMAND} --build . --target test_${name} --config $<CONFIGURATION> )
	endif()

	# Validator tests (warnings)
	string(REGEX REPLACE ".*(validator\\..*)\\.qasm$" "\\1" name ${filename})
	if(NOT ${name} STREQUAL ${filename})
		add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed ${CMAKE_CURRENT_BINARY_DIR}/${name}.check ${CMAKE_CURRENT_BINARY_DIR}/${name}.out
			COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/ignoreResult.pl $<TARGET_FILE:vc4asm> -V -Q -i vc4.qinc ${filename} > ${CMAKE_CURRENT_BINARY_DIR}/${name}.out 2>&1
			COMMAND perl ${CMAKE_CURRENT_SOURCE_DIR}/checkOutput.pl ${filename} < ${CMAKE_CURRENT_BINARY_DIR}/${name}.out > ${CMAKE_CURRENT_BINARY_DIR}/${name}.check
			COMMAND cat ${CMAKE_CURRENT_BINARY_DIR}/${name}.check 2> ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed # Work around for cmake bug #15474
			DEPENDS vc4asm ${filename} ${CMAKE_CURRENT_SOURCE_DIR}/checkOutput.pl ${CMAKE_CURRENT_SOURCE_DIR}/ignoreResult.pl VERBATIM )
		add_custom_target( test_${name} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${name}.passed )
		list(APPEND test_validator test_${name} )
		add_test(NAME ${name} WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
			COMMAND ${CMAKE_COMMAND} --build . --target test_${name} --config $<CONFIGURATION> )
	endif()
endforeach()

# Test groups
add_custom_target( test_qasm DEPENDS ${test_qasm} )
add_custom_target( test_cout DEPENDS ${test_cout} )
add_custom_target( test_parser DEPENDS ${test_parser} )
add_custom_target( test_validator DEPENDS ${test_validator} )
# Invoking cmake --build for every test case as work around to build non executable prereqisites
# is extraordinary slow on a Raspberry Pi.
# So let's skip Cmakes test environment and add a test target manually.
add_custom_target( test_all ALL DEPENDS test_qasm test_cout test_parser test_validator )
