cmake_minimum_required(VERSION 2.8.9...3.22)
project(go-rapidyenc)

option(BUILD_NATIVE "Optimise for and target only the build host's CPU; this build may not be re-distributable" OFF)
option(DISABLE_AVX256 "Disable the use of 256-bit AVX instructions on x86 processors" OFF)

include(CheckCXXCompilerFlag)
include(CheckIncludeFileCXX)
include(CheckCXXSymbolExists)

set(CMAKE_MACOSX_RPATH OFF)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release)
endif()
if(NOT TARGET_ARCH)
	if(CMAKE_GENERATOR_PLATFORM)
		set(TARGET_ARCH ${CMAKE_GENERATOR_PLATFORM})
	else()
		set(TARGET_ARCH ${CMAKE_SYSTEM_PROCESSOR})
	endif()
endif()

if (${TARGET_ARCH} MATCHES "i386|i686|x86|x86_64|x64|amd64|AMD64|win32|Win32")
	set(IS_X86 TRUE)
	if(NOT MSVC)
		CHECK_CXX_SYMBOL_EXISTS(__ILP32__ "" IS_X32)
	endif()
endif()
if (${TARGET_ARCH} MATCHES "arm|ARM")
	set(IS_ARM32 TRUE)
endif()
if (${TARGET_ARCH} MATCHES "aarch64|arm64|ARM64")
	set(IS_ARM64 TRUE)
endif()

if(DISABLE_AVX256)
	add_compile_definitions(YENC_DISABLE_AVX256=1)
endif()

if(NOT MSVC)
	if(BUILD_NATIVE)
		CHECK_CXX_COMPILER_FLAG("-march=native" COMPILER_SUPPORTS_NATIVE)
		if(COMPILER_SUPPORTS_NATIVE)
			add_compile_options(-march=native)
			add_compile_definitions(YENC_BUILD_NATIVE=1)
		endif()
	endif()
	if(NOT BUILD_NATIVE AND IS_X86)
		CHECK_CXX_COMPILER_FLAG("-mno-avx256-split-unaligned-load -mno-avx256-split-unaligned-store" COMPILER_SUPPORTS_NOSPLIT_AVX256)
		if(COMPILER_SUPPORTS_NOSPLIT_AVX256)
			add_compile_options(-mno-avx256-split-unaligned-load -mno-avx256-split-unaligned-store)
		endif()
	endif()
endif()

if(IS_ARM64)
	set(DECODER_NEON_FILE decoder_neon64.cc)
else()
	set(DECODER_NEON_FILE decoder_neon.cc)
endif()

set(SRC_DIR ./rapidyenc/src)
set(CRCUTIL_DIR ./rapidyenc/crcutil-1.0)
set(RAPIDYENC_SOURCES
	${SRC_DIR}/platform.cc
	${SRC_DIR}/encoder.cc
	${SRC_DIR}/decoder.cc
	${SRC_DIR}/crc.cc
	
	${SRC_DIR}/encoder_sse2.cc
	${SRC_DIR}/decoder_sse2.cc
	${SRC_DIR}/encoder_ssse3.cc
	${SRC_DIR}/decoder_ssse3.cc
	${SRC_DIR}/encoder_avx.cc
	${SRC_DIR}/decoder_avx.cc
	${SRC_DIR}/encoder_avx2.cc
	${SRC_DIR}/decoder_avx2.cc
	${SRC_DIR}/encoder_vbmi2.cc
	${SRC_DIR}/decoder_vbmi2.cc
	${SRC_DIR}/encoder_neon.cc
	${SRC_DIR}/${DECODER_NEON_FILE}
	${SRC_DIR}/crc_folding.cc
	${SRC_DIR}/crc_folding_256.cc
	${SRC_DIR}/crc_arm.cc
)
set(CRCUTIL_SOURCES 
	${CRCUTIL_DIR}/code/crc32c_sse4.cc 
	${CRCUTIL_DIR}/code/multiword_64_64_cl_i386_mmx.cc 
	${CRCUTIL_DIR}/code/multiword_64_64_gcc_amd64_asm.cc 
	${CRCUTIL_DIR}/code/multiword_64_64_gcc_i386_mmx.cc 
	${CRCUTIL_DIR}/code/multiword_64_64_intrinsic_i386_mmx.cc 
	${CRCUTIL_DIR}/code/multiword_128_64_gcc_amd64_sse2.cc 
	${CRCUTIL_DIR}/examples/interface.cc
)

include_directories(${CRCUTIL_DIR}/code ${CRCUTIL_DIR}/examples ${CRCUTIL_DIR}/tests)
add_compile_definitions(CRCUTIL_USE_MM_CRC32=0)

if(MSVC)
	add_compile_options(/W2)
	# TODO: this doesn't exclude the flags in the 'debug' build config in MSVC
	if(NOT ${CMAKE_BUILD_TYPE} MATCHES "Debug")
		add_compile_options(/GS- /Gy /sdl- /Oy /Oi)
		add_link_options(/OPT:REF /OPT:ICF)
	endif()
else()
	add_compile_options(-Wall -Wextra -Wno-unused-function -fno-exceptions -fwrapv -Wno-expansion-to-defined -fvisibility=hidden)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
	if(NOT ${CMAKE_BUILD_TYPE} MATCHES "Debug")
		add_compile_options(-fomit-frame-pointer)
	endif()
endif()

add_library(rapidyenc OBJECT ${RAPIDYENC_SOURCES} ${CRCUTIL_SOURCES})
set_property(TARGET rapidyenc PROPERTY POSITION_INDEPENDENT_CODE 1)  # needed for shared build

if(MSVC)
	if(IS_X86)
		set_source_files_properties(${SRC_DIR}/encoder_avx.cc PROPERTIES COMPILE_OPTIONS /arch:AVX)
		set_source_files_properties(${SRC_DIR}/decoder_avx.cc PROPERTIES COMPILE_OPTIONS /arch:AVX)
		set_source_files_properties(${SRC_DIR}/encoder_avx2.cc PROPERTIES COMPILE_OPTIONS /arch:AVX2)
		set_source_files_properties(${SRC_DIR}/decoder_avx2.cc PROPERTIES COMPILE_OPTIONS /arch:AVX2)
		set_source_files_properties(${SRC_DIR}/encoder_vbmi2.cc PROPERTIES COMPILE_OPTIONS /arch:AVX512)
		set_source_files_properties(${SRC_DIR}/decoder_vbmi2.cc PROPERTIES COMPILE_OPTIONS /arch:AVX512)
		set_source_files_properties(${SRC_DIR}/crc_folding_256.cc PROPERTIES COMPILE_OPTIONS /arch:AVX2)
	endif()
else()
	if(IS_X86)
		set_source_files_properties(${SRC_DIR}/encoder_sse2.cc PROPERTIES COMPILE_OPTIONS -msse2)
		set_source_files_properties(${SRC_DIR}/decoder_sse2.cc PROPERTIES COMPILE_OPTIONS -msse2)
		set_source_files_properties(${SRC_DIR}/encoder_ssse3.cc PROPERTIES COMPILE_OPTIONS -mssse3)
		set_source_files_properties(${SRC_DIR}/decoder_ssse3.cc PROPERTIES COMPILE_OPTIONS -mssse3)
		set_source_files_properties(${SRC_DIR}/encoder_avx.cc PROPERTIES COMPILE_OPTIONS "-mavx;-mpopcnt")
		set_source_files_properties(${SRC_DIR}/decoder_avx.cc PROPERTIES COMPILE_OPTIONS "-mavx;-mpopcnt")
		set_source_files_properties(${SRC_DIR}/encoder_avx2.cc PROPERTIES COMPILE_OPTIONS "-mavx2;-mpopcnt;-mbmi;-mbmi2;-mlzcnt")
		set_source_files_properties(${SRC_DIR}/decoder_avx2.cc PROPERTIES COMPILE_OPTIONS "-mavx2;-mpopcnt;-mbmi;-mbmi2;-mlzcnt")
		CHECK_CXX_COMPILER_FLAG("-mavx512vbmi2 -mavx512vl -mavx512bw" COMPILER_SUPPORTS_VBMI2)
		if(COMPILER_SUPPORTS_VBMI2)
			set_source_files_properties(${SRC_DIR}/encoder_vbmi2.cc PROPERTIES COMPILE_OPTIONS "-mavx512vbmi2;-mavx512vl;-mavx512bw;-mpopcnt;-mbmi;-mbmi2;-mlzcnt")
			set_source_files_properties(${SRC_DIR}/decoder_vbmi2.cc PROPERTIES COMPILE_OPTIONS "-mavx512vbmi2;-mavx512vl;-mavx512bw;-mpopcnt;-mbmi;-mbmi2;-mlzcnt")
		endif()
		set_source_files_properties(${SRC_DIR}/crc_folding.cc PROPERTIES COMPILE_OPTIONS "-mssse3;-msse4.1;-mpclmul")
		CHECK_CXX_COMPILER_FLAG("-mvpclmulqdq" COMPILER_SUPPORTS_VPCLMULQDQ)
		if(COMPILER_SUPPORTS_VPCLMULQDQ)
			set_source_files_properties(${SRC_DIR}/crc_folding_256.cc PROPERTIES COMPILE_OPTIONS "-mavx2;-mvpclmulqdq;-mpclmul")
		endif()
		
		if(IS_X32)
			add_compile_definitions(CRCUTIL_USE_ASM=0)
		endif()
	endif()
	
	if(NOT APPLE) # M1 Macs don't seem to need these ARM options
		if(IS_ARM32)
			set_source_files_properties(${SRC_DIR}/encoder_neon.cc PROPERTIES COMPILE_OPTIONS -mfpu=neon)
			set_source_files_properties(${SRC_DIR}/${DECODER_NEON_FILE} PROPERTIES COMPILE_OPTIONS -mfpu=neon)
			set_source_files_properties(${SRC_DIR}/crc_arm.cc PROPERTIES COMPILE_OPTIONS "-march=armv8-a+crc;-mfpu=fp-armv8")
		endif()
		if(IS_ARM64)
			set_source_files_properties(${SRC_DIR}/crc_arm.cc PROPERTIES COMPILE_OPTIONS -march=armv8-a+crc)
		endif()
		
		CHECK_INCLUDE_FILE_CXX("sys/auxv.h" COMPILER_HAS_AUXV)
		if(NOT COMPILER_HAS_AUXV)
			add_compile_definitions(MISSING_AUXV=1)
		endif()
	endif()
endif()


add_library(rapidyenc_shared SHARED $<TARGET_OBJECTS:rapidyenc> ./rapidyenc/rapidyenc.c)
target_compile_definitions(rapidyenc_shared PUBLIC BUILD_SHARED=1)
# suggestion from https://stackoverflow.com/a/67027398/459150
set_target_properties(rapidyenc_shared PROPERTIES OUTPUT_NAME rapidyenc ARCHIVE_OUTPUT_DIRECTORY $<0:>)
set_target_properties(rapidyenc_shared PROPERTIES OUTPUT_NAME rapidyenc RUNTIME_OUTPUT_DIRECTORY $<0:>)
