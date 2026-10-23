cmake_minimum_required(VERSION 3.20)
project(AudioCleanHot VERSION 3.0.0 LANGUAGES C CXX)

# AudioClean Hot - Apex V3: biblioteca nativa audioclean_ultra.dll
#   cmake -B build -DCMAKE_BUILD_TYPE=Release
#   cmake --build build --config Release

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Tipo de build" FORCE)
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# CRT estatico: a DLL carrega em qualquer Windows sem o VC++ Redistributable.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

option(ACH_ENABLE_CUDA "Compila os kernels CUDA quando o toolkit estiver disponivel" ON)

set(ACH_CU_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/backend/src/audioclean_master.cu
  ${CMAKE_CURRENT_SOURCE_DIR}/backend/src/video_temporal_inpaint.cu
  ${CMAKE_CURRENT_SOURCE_DIR}/backend/src/auto_watermark_detector.cu)

# ---------------------------------------------------------------------------------------------
# Deteccao de CUDA. Sem toolkit (ou sem integracao com o gerador), os .cu sao compilados como C++20
# puro: o caminho de CPU (FFT Radix-2 + SSE2/AVX2/AVX-512 + multithread) esta dentro dos proprios
# arquivos (#if defined(ACH_GPU_COMPILER) vs. fallback), entao a compilacao nunca falha.
include(CheckLanguage)
set(ACH_CUDA_ACTIVE OFF)
if(ACH_ENABLE_CUDA)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
      set(CMAKE_CUDA_ARCHITECTURES 75 86 89)
    endif()
    enable_language(CUDA)
    find_package(CUDAToolkit QUIET)
    if(CUDAToolkit_FOUND)
      set(ACH_CUDA_ACTIVE ON)
    endif()
  endif()
endif()

# HIP (AMD ROCm / HIP SDK para Windows): EXPERIMENTAL, desligado por padrao e nao validado (sem GPU AMD nem HIP
# SDK na maquina de desenvolvimento). As chamadas cuda*/cufft* sao mapeadas em backend/include/gpu_portability.hpp.
# O caminho AMD/Intel validado e o DirectX 12 compute (d3d12_compute.cpp), sempre compilado.
option(ACH_ENABLE_HIP "Compila os kernels de GPU para AMD via HIP (EXPERIMENTAL, requer HIP SDK)" OFF)
set(ACH_HIP_ACTIVE OFF)
if(ACH_ENABLE_HIP AND NOT ACH_CUDA_ACTIVE)
  check_language(HIP)
  if(CMAKE_HIP_COMPILER)
    enable_language(HIP)
    find_package(hip QUIET)
    find_package(hipfft QUIET)
    if(hip_FOUND AND hipfft_FOUND)
      set(ACH_HIP_ACTIVE ON)
    endif()
  endif()
  if(NOT ACH_HIP_ACTIVE)
    message(WARNING "AudioClean Hot: ACH_ENABLE_HIP=ON, mas o HIP SDK (hip + hipfft) nao foi encontrado - build sem HIP")
  endif()
endif()

if(ACH_CUDA_ACTIVE)
  if(MSVC AND MSVC_VERSION LESS 1930)
    # O nvcc ignora -std=c++20 com o host MSVC 2019 (v142); os .cu sao escritos em C++17 compativel.
    set(CMAKE_CUDA_STANDARD 17)
  else()
    set(CMAKE_CUDA_STANDARD 20)
  endif()
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)
  message(STATUS "AudioClean Hot: CUDA ${CUDAToolkit_VERSION} ativo (arquiteturas ${CMAKE_CUDA_ARCHITECTURES}, C++${CMAKE_CUDA_STANDARD})")
elseif(ACH_HIP_ACTIVE)
  set_source_files_properties(${ACH_CU_SOURCES} PROPERTIES LANGUAGE HIP)
  message(STATUS "AudioClean Hot: HIP ativo (EXPERIMENTAL, nao validado)")
else()
  set_source_files_properties(${ACH_CU_SOURCES} PROPERTIES LANGUAGE CXX)
  if(MSVC)
    set_source_files_properties(${ACH_CU_SOURCES} PROPERTIES COMPILE_OPTIONS "/TP")
  else()
    set_source_files_properties(${ACH_CU_SOURCES} PROPERTIES COMPILE_OPTIONS "-xc++")
  endif()
  message(STATUS "AudioClean Hot: CUDA indisponivel - .cu compilados como C++20 (motor CPU SIMD + multithread)")
endif()

add_library(audioclean_ultra SHARED
  backend/src/hardware_profiler.cpp
  backend/src/d3d12_compute.cpp
  ${ACH_CU_SOURCES}
  backend/include/audioclean_api.h
  backend/include/hardware_profiler.hpp
  backend/include/d3d12_compute.hpp
  backend/include/gpu_portability.hpp
  backend/include/auto_watermark_detector.hpp
  backend/include/video_temporal_inpaint.hpp)

target_include_directories(audioclean_ultra PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/backend/include)
target_compile_definitions(audioclean_ultra PRIVATE AUDIOCLEAN_EXPORTS NOMINMAX WIN32_LEAN_AND_MEAN _CRT_SECURE_NO_WARNINGS)

if(MSVC)
  # Sem /arch:AVX*: o baseline e SSE2 (todo x64). AVX2/FMA3/AVX-512 so por despacho em runtime
  # apos CPUID + XGETBV -> nenhuma STATUS_ILLEGAL_INSTRUCTION (0xC000001D) em CPUs legadas.
  target_compile_options(audioclean_ultra PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>:/W3 /EHsc /utf-8 /permissive- /Zc:__cplusplus /MP>
    $<$<COMPILE_LANGUAGE:CUDA>:--use_fast_math -Xcompiler=/utf-8 -Xcompiler=/EHsc -Xcompiler=/W3>)
else()
  target_compile_options(audioclean_ultra PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>:-Wall -O2>
    $<$<COMPILE_LANGUAGE:CUDA>:--use_fast_math>)
endif()

target_link_libraries(audioclean_ultra PRIVATE psapi dxgi)

if(ACH_CUDA_ACTIVE)
  target_compile_definitions(audioclean_ultra PRIVATE ACH_HAS_CUDA=1)
  set_target_properties(audioclean_ultra PROPERTIES CUDA_RUNTIME_LIBRARY Shared CUDA_SEPARABLE_COMPILATION OFF)
  target_link_libraries(audioclean_ultra PRIVATE CUDA::cudart CUDA::cufft)
  # DLLs CUDA com carregamento tardio: a audioclean_ultra.dll abre mesmo sem o toolkit instalado e
  # so usa a GPU depois que o runtime for encontrado e o auto-teste de paridade passar.
  file(GLOB _ach_cudart_dlls "${CUDAToolkit_BIN_DIR}/cudart64_*.dll" "${CUDAToolkit_BIN_DIR}/x64/cudart64_*.dll")
  file(GLOB _ach_cufft_dlls "${CUDAToolkit_BIN_DIR}/cufft64_*.dll" "${CUDAToolkit_BIN_DIR}/x64/cufft64_*.dll")
  if(MSVC AND _ach_cudart_dlls AND _ach_cufft_dlls)
    list(GET _ach_cudart_dlls 0 _ach_cudart_dll)
    list(GET _ach_cufft_dlls 0 _ach_cufft_dll)
    get_filename_component(_ach_cudart_name "${_ach_cudart_dll}" NAME_WE)
    get_filename_component(_ach_cufft_name "${_ach_cufft_dll}" NAME_WE)
    target_link_options(audioclean_ultra PRIVATE "/DELAYLOAD:${_ach_cudart_name}.dll" "/DELAYLOAD:${_ach_cufft_name}.dll")
    target_link_libraries(audioclean_ultra PRIVATE delayimp)
    target_compile_definitions(audioclean_ultra PRIVATE
      ACH_CUDART_DLL_BASENAME=${_ach_cudart_name}
      ACH_CUFFT_DLL_BASENAME=${_ach_cufft_name})
    message(STATUS "AudioClean Hot: delay-load de ${_ach_cudart_name}.dll e ${_ach_cufft_name}.dll")
  endif()
elseif(ACH_HIP_ACTIVE)
  target_compile_definitions(audioclean_ultra PRIVATE ACH_HAS_CUDA=0 ACH_USE_HIP=1)
  target_link_libraries(audioclean_ultra PRIVATE hip::host hip::hipfft)
else()
  target_compile_definitions(audioclean_ultra PRIVATE ACH_HAS_CUDA=0)
endif()

set_target_properties(audioclean_ultra PROPERTIES
  OUTPUT_NAME audioclean_ultra
  PREFIX ""
  RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/Release"
  RUNTIME_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/Debug")
