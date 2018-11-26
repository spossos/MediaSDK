# Copyright (c) 2018 Intel Corporation
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

if(MFX_ENABLE_KERNELS)
  mfx_include_dirs( )
  set( MSDK_GENX_ROOT ${MSDK_LIB_ROOT}/genx )

  if (BUILD_KERNELS)
    # Here we define build steps to generate c-array binary kernels. If you
    # don't use BUILD_KERNELS options these kernels will just be used from
    # pre-built form. If you will regenerate kernels you may notice the
    # difference from the per-built kernels in git-diff.

    # Helper application to generate c-array data representation of .isa binary
    # kernels.
    add_executable(embed_isa ${MSDK_GENX_ROOT}/mctf/src/embed_isa.c)
    set_target_properties(embed_isa PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/mctf)

    # Gets gen version from the platform name
    # For example, for ${platform}=skl it will return ${genx}=gen9
    function(get_genx platform genx)
      if(${platform} STREQUAL "bdw")
        set(genx "gen8" PARENT_SCOPE)
      elseif(${platform} STREQUAL "skl")
        set(genx "gen9" PARENT_SCOPE)
      elseif(${platform} STREQUAL "icl")
        set(genx "gen11" PARENT_SCOPE)
      elseif(${platform} STREQUAL "icllp")
        set(genx "gen11lp" PARENT_SCOPE)
      endif()
    endfunction()

    # Generates CM kernel suitable for usage in C/C++ application sources
    #   ${src} - input CM kernel source file
    #   ${platform} - platform to generate kernel for
    #   ${out_dir} - directory to generate output
    # Generated outputs for ${src}=path/file.cpp, ${platform}=skl:
    #   path/include/file_skl_isa.h
    #   path/src/file_skl_isa.cpp
    function(add_cmc_target src platform out_dir)
      get_filename_component(tgt ${src} NAME_WE)
      get_genx(genx ${platform})
      add_custom_command(
        OUTPUT ${tgt}_${platform}.isa
        DEPENDS embed_isa ${src}
        COMMAND ${CMC}
          -c -Qxcm -Qxcm_release -Qxcm_print_asm_count
          -menableiga -mCM_init_global -mCM_printregusage
          -Dtarget_${genx} -Qxcm_jit_target=${genx}
          -isystem ${CMC_INCLUDE}
          ${src} -o ${tgt}_${platform}.isa)
      add_custom_command(
        OUTPUT ${tgt}_${platform}.cpp ${tgt}_${platform}.h
        DEPENDS embed_isa ${tgt}_${platform}.isa
        COMMAND embed_isa ${tgt}_${platform}.isa)
      add_custom_command(
        OUTPUT ${out_dir}/include/${tgt}_${platform}_isa.h ${out_dir}/src/${tgt}_${platform}_isa.cpp
        DEPENDS ${tgt}_${platform}.cpp ${tgt}_${platform}.h
        COMMAND ${CMAKE_COMMAND} -E copy ${tgt}_${platform}.h ${out_dir}/include/${tgt}_${platform}_isa.h
        COMMAND ${CMAKE_COMMAND} -E copy ${tgt}_${platform}.cpp ${out_dir}/src/${tgt}_${platform}_isa.cpp)
    endfunction()
  endif()

  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/asc/include )
  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/copy_kernels/include )
  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/field_copy/include )
  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/h264_encode/include )
  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/mctf/include )
  include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/../cmrt_cross_platform/include )

  set( defs "" )
  set( sources "" )
  set( sources.plus "" )

  set( prefix ${MSDK_GENX_ROOT}/asc/src )
  list( APPEND sources
      ${prefix}/genx_scd_bdw_isa.cpp
      ${prefix}/genx_scd_cnl_isa.cpp
      ${prefix}/genx_scd_icl_isa.cpp
      ${prefix}/genx_scd_icllp_isa.cpp
      ${prefix}/genx_scd_skl_isa.cpp
  )

  set( prefix ${MSDK_GENX_ROOT}/copy_kernels/src )
  list( APPEND sources
    ${prefix}/genx_cht_copy_isa.cpp
    ${prefix}/genx_skl_copy_isa.cpp
    ${prefix}/genx_icl_copy_isa.cpp
    ${prefix}/genx_icllp_copy_isa.cpp
    ${prefix}/genx_cnl_copy_isa.cpp
  )

  set( prefix ${MSDK_GENX_ROOT}/field_copy/src )
  list( APPEND sources
    ${prefix}/genx_fcopy_gen8_isa.cpp
    ${prefix}/genx_fcopy_gen9_isa.cpp
    ${prefix}/genx_fcopy_gen10_isa.cpp
    ${prefix}/genx_fcopy_gen11_isa.cpp
    ${prefix}/genx_fcopy_gen11lp_isa.cpp
  )

  set( prefix ${MSDK_GENX_ROOT}/h264_encode/src )
  list( APPEND sources
    ${prefix}/genx_bdw_simple_me_isa.cpp
    ${prefix}/genx_cnl_histogram_isa.cpp
    ${prefix}/genx_cnl_simple_me_isa.cpp
    ${prefix}/genx_hsw_simple_me_isa.cpp
    ${prefix}/genx_hsw_simple_me_proto.cpp
    ${prefix}/genx_icl_histogram_isa.cpp
    ${prefix}/genx_icl_simple_me_isa.cpp
    ${prefix}/genx_icllp_histogram_isa.cpp
    ${prefix}/genx_icllp_simple_me_isa.cpp
    ${prefix}/genx_skl_histogram_isa.cpp
    ${prefix}/genx_skl_simple_me_isa.cpp
  )

  if(MFX_ENABLE_MCTF)
    if(BUILD_KERNELS)
      foreach(platform bdw skl icl icllp)
        add_cmc_target(${MSDK_GENX_ROOT}/mctf/src/genx_me.cpp ${platform} ${MSDK_GENX_ROOT}/mctf)
        add_cmc_target(${MSDK_GENX_ROOT}/mctf/src/genx_mc.cpp ${platform} ${MSDK_GENX_ROOT}/mctf)
        add_cmc_target(${MSDK_GENX_ROOT}/mctf/src/genx_sd.cpp ${platform} ${MSDK_GENX_ROOT}/mctf)
      endforeach()
    endif()

    set( prefix ${MSDK_GENX_ROOT}/mctf/src )
    list( APPEND sources
      ${prefix}/genx_mc_bdw_isa.cpp
      ${prefix}/genx_mc_icl_isa.cpp
      ${prefix}/genx_mc_icllp_isa.cpp
      ${prefix}/genx_mc_skl_isa.cpp
      ${prefix}/genx_me_bdw_isa.cpp
      ${prefix}/genx_me_icl_isa.cpp
      ${prefix}/genx_me_icllp_isa.cpp
      ${prefix}/genx_me_skl_isa.cpp
      ${prefix}/genx_sd_bdw_isa.cpp
      ${prefix}/genx_sd_icl_isa.cpp
      ${prefix}/genx_sd_icllp_isa.cpp
      ${prefix}/genx_sd_skl_isa.cpp
    )
  endif()

  make_library(genx none static )
  set( defs "" )
endif()
