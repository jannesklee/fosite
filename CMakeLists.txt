# CMake project file for FOSITE
cmake_minimum_required (VERSION 2.8.10)
project (FOSITE NONE)

# detect git
find_package(Git)
if(Git_FOUND)
  # get the latest git version tag and commit hash
  # adds "-dirty" if working tree is not clean
  execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --tag --dirty
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  add_definitions("-DVERSION=\"${GIT_VERSION}\"")
else()
  add_definitions("-DVERSION=\"0.8.2\"") 
endif()

# make sure that the default is a RELEASE
if (NOT CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE RELEASE CACHE STRING
      "Choose the type of build, options are: None Debug Release Profiling."
      FORCE)
endif (NOT CMAKE_BUILD_TYPE)

# default installation
get_filename_component (default_prefix ".." ABSOLUTE)
set (CMAKE_INSTALL_PREFIX ${default_prefix} CACHE STRING
      "Choose the installation directory; by default it installs in the build directory."
      FORCE)

#------------------------ command line options -------------------------#
option(PARALLEL "PARALLEL" OFF)
option(MPI_USE_SENDRECV "MPI_USE_SENDRECV" ON)
option(USE_MPI_MOD "USE_MPI_MOD" ON)
if (PARALLEL MATCHES "ON")
  message(STATUS "Configuring parallel build using MPI")
  add_definitions(-DPARALLEL)
  if(USE_MPI_MOD MATCHES "ON")
    add_definitions(-DHAVE_MPI_MOD)
  else()
    add_definitions(-DHAVE_MPIF_H)
  endif()
  if(MPI_USE_SENDRECV MATCHES "ON")
    add_definitions(-DMPI_USE_SENDRECV)
  endif()
  if (CMAKE_SYSTEM_NAME MATCHES "SX-Aurora")
    set(CMAKE_Fortran_COMPILER ${MPI_Fortran_COMPILER})
  else()
    set(CMAKE_Fortran_COMPILER mpifort)
    message(WARNING
      "There is a known problem causing segfaults with OpenMPI \
      (experienced with version 3.1.3), depending on the amount of processes \
      used. If you experience any problems with OpenMPI \
      please consider to try another version or the MPICH implementation.")
  endif()
endif()

enable_language (Fortran)
# workaround for unknown compiler ID NEC
if (CMAKE_Fortran_COMPILER MATCHES "(.*)nfort")
  message(STATUS "Fortran compiler ID set to NEC")
  set(CMAKE_Fortran_COMPILER_ID "NEC")
endif()
enable_testing()

set (X86_VECTOR_EXTENSION NONE CACHE STRING "Enable x86 vector extensions, possible values are: NONE AVX AVX2 AVX-512")

#-------------------- choose compiler & set flags ---------------------#
#get_filename_component (Fortran_COMPILER_NAME ${CMAKE_Fortran_COMPILER} NAME)
set (CMAKE_Fortran_FLAGS "${FCFLAGS}")
if (CMAKE_Fortran_COMPILER_ID MATCHES "GNU")
  # GNU compiler, i.e. gfortran
  add_compile_options(-fdefault-real-8 -cpp -ffree-line-length-none)
  if (NOT X86_VECTOR_EXTENSION MATCHES "NONE")
    message(STATUS "using x86 advanced vector instructions: " ${X86_VECTOR_EXTENSION})
    if (X86_VECTOR_EXTENSION MATCHES "AVX")
      add_compile_options(-mavx)
    elseif (X86_VECTOR_EXTENSION MATCHES "AVX2")
      add_compile_options(-mavx2)
    elseif (X86_VECTOR_EXTENSION MATCHES "AVX-512")
      add_compile_options(-mavx512f)
    endif()
  endif()
  set (CMAKE_Fortran_FLAGS_RELEASE "-funroll-all-loops -fno-f2c -O3")
  set (CMAKE_Fortran_FLAGS_PROFILING "-funroll-all-loops -fno-f2c -O3 -pg -no-pie")
  set (CMAKE_Fortran_FLAGS_DEBUG   "-fno-f2c -O2 -g -fcheck=all -Wunused")
  if(CMAKE_Fortran_COMPILER_VERSION VERSION_LESS 6.2)
    message(FATAL_ERROR "gfortran must be at least version 6.2")
  endif()
elseif (CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
  # intel Fortran compiler, i.e. ifort
  add_compile_options(-cpp -r8 -stand f03)
  set (CMAKE_Fortran_FLAGS_RELEASE "-O3")
  set (CMAKE_Fortran_FLAGS_DEBUG   "-O0 -g")
  set (CMAKE_Fortran_FLAGS_PROFILING "-O3 -pg")
elseif (CMAKE_Fortran_COMPILER_ID MATCHES "NEC")
  add_compile_options(-report-all -proginf -fdefault-real=8 -x f95-cpp-input)
  set (CMAKE_Fortran_FLAGS_RELEASE "-finline-max-depth=5 -finline-max-function-size=150 -finline-functions -floop-collapse")
  set (CMAKE_Fortran_FLAGS_DEBUG   "-O1 -g -fcheck=all -finline-max-depth=5 -finline-functions")
  set (CMAKE_Fortran_FLAGS_PROFILING "-finline-max-depth=5 -finline-max-function-size=150 -finline-functions -floop-collapse -ftrace")
  set (CMAKE_EXE_LINKER_FLAGS "-proginf")
  set (CMAKE_EXE_LINKER_FLAGS_PROFILING "-ftrace")
#  set (LINK_FLAGS "")
else (Fortran_COMPILER_NAME_ID MATCHES "GNU")
  # unknown compiler
  message (STATUS "Found unsupported Fortran compiler: " ${CMAKE_Fortran_COMPILER})
  set (CMAKE_Fortran_FLAGS_RELEASE "")
  set (CMAKE_Fortran_FLAGS_DEBUG   "")
  set (CMAKE_Fortran_FLAGS_PROFILING "")
endif (CMAKE_Fortran_COMPILER_ID MATCHES "GNU")

# set preprocessor macro to the vector length of the target system
# DEFAULT: 1
if (CMAKE_SYSTEM_NAME MATCHES "(.*)SX-Aurora")
  add_definitions(-DVECTOR_LENGTH="256" -DNECSXAURORA)
else ()#(CMAKE_SYSTEM_NAME MATCHES "(.*)SX-AURORA")
  if (X86_VECTOR_EXTENSION MATCHES "AVX")
    add_definitions(-DVECTOR_LENGTH="16")
  elseif (X86_VECTOR_EXTENSION MATCHES "AVX2")
    add_definitions(-DVECTOR_LENGTH="16")
  elseif (X86_VECTOR_EXTENSION MATCHES "AVX-512")
    add_definitions(-DVECTOR_LENGTH="32")
  else()
    add_definitions(-DVECTOR_LENGTH="1")
  endif()
endif ()#(CMAKE_SYSTEM_NAME MATCHES "(.*)SX-AURORA")

MARK_AS_ADVANCED(CMAKE_Fortran_FLAGS_PROFILING
    CMAKE_EXE_LINKER_FLAGS_PROFILING)


#----------------------- search GSL ------------------------------------------#
option(USE_GSL "Add GSL/FGSL support" OFF)
if(USE_GSL MATCHES ON)
  find_package(GSL)
else()
  set(GSL_FOUND CACHE BOOL FALSE)
endif()
if(GSL_FOUND)
  set(FGSL_SOURCE_DIR ${CMAKE_SOURCE_DIR}/contrib/fgsl) 
  set(FGSL_BINARY_DIR ${CMAKE_BINARY_DIR}/contrib/fgsl) 
  include_directories(${GSL_INCLUDE_DIRS} ${FGSL_BINARY_DIR}/include/fgsl)
  # add gsl fortran interface as external project
  if(NOT EXISTS "${FGSL_SOURCE_DIR}/configure.ac")
    message(STATUS "fgsl: found empty source dir -> fetching code")
    execute_process(
        COMMAND git submodule update --init --force contrib/fgsl
	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
  endif()
  if(NOT EXISTS ${FGSL_SOURCE_DIR}/configure)
    if(NOT EXISTS ${FGSL_SOURCE_DIR}/m4)
      file(MAKE_DIRECTORY ${FGSL_SOURCE_DIR}/m4)
    endif()
    execute_process(
      COMMAND autoreconf -i
      WORKING_DIRECTORY ${FGSL_SOURCE_DIR}
    )
  endif()
  include(ExternalProject)
  ExternalProject_Add(fgsl
    PREFIX ${FGSL_BINARY_DIR}
    SOURCE_DIR ${FGSL_BINARY_DIR}/src
    BUILD_IN_SOURCE 0
    BINARY_DIR ${FGSL_BINARY_DIR}/build
    INSTALL_DIR ${FGSL_BINARY_DIR}
    DOWNLOAD_DIR ${FGSL_BINARY_DIR}
    STAMP_DIR ${FGSL_BINARY_DIR}/stamp
    TMP_DIR ${FGSL_BINARY_DIR}/tmp
    DOWNLOAD_COMMAND  ${CMAKE_COMMAND} -E remove_directory "${FGSL_BINARY_DIR}/src" && ${CMAKE_COMMAND} -E create_symlink "${FGSL_SOURCE_DIR}" "${FGSL_BINARY_DIR}/src"
    CONFIGURE_COMMAND ../src/configure -q --prefix=${FGSL_BINARY_DIR}
    BUILD_COMMAND make
    INSTALL_COMMAND make install)
  add_definitions(-DHAVE_FGSL)
endif()

#----------------------- search FFTW -----------------------------------------#
set(CMAKE_FIND_ROOT_PATH ${FFTW_DIR})

# search for library
find_library(
  FFTW_LIB
	NAMES fftw3 aslfftw3
	DOC "Fastest Fourier Transform of the West."
  HINTS /opt/nec/ve/nlc/1.0.0/musl/lib/ ${FFTW_DIR}/lib
  HINTS /opt/nec/ve/nlc/1.0.0/lib/ ${FFTW_DIR}/lib
)

if(PARALLEL)
  find_library(
    FFTWMPI_LIB
	  NAMES fftw3_mpi aslfftw3_mpi
    DOC "Distributed parallel implementation of FFTW."
    HINTS /opt/nec/ve/nlc/1.0.0/musl/lib/ ${FFTW_DIR}/lib
    HINTS /opt/nec/ve/nlc/1.0.0/lib/ ${FFTW_DIR}/lib
  )
endif()

#end search for libraries

# search for headers
if(NOT PARALLEL)
  set (fftw_incfiles fftw3.f03)
elseif(PARALLEL)
  set (fftw_incfiles fftw3-mpi.f03)
endif()

find_path(
  FFTW_INCLUDE
  NAMES ${fftw_incfiles}
)
if (CMAKE_SYSTEM_NAME MATCHES "(.*)SX-Aurora")
  set (FFTW_INCLUDE /opt/nec/ve/nlc/1.0.0/include/)
  if (NOT PARALLEL)
    set (ASL_INCLUDE /opt/nec/ve/nlc/1.0.0/include/mod)
  elseif (PARALLEL)
    set (ASL_INCLUDE /opt/nec/ve/nlc/1.0.0/include/mod_mpi)
  endif()
  message(STATUS "ASL headers: ${ASL_INCLUDE}")
endif()
# end search for headers

if(NOT FFTW_LIB)
  message(WARNING "No FFTW Library found.")
else()
  if(NOT FFTW_INCLUDE)
    message(WARNING "No FFTW header file found.")
  endif()
  message(STATUS "FFTW library: ${FFTW_LIB}")
  if(PARALLEL)
    message(STATUS "FFTW MPI library: ${FFTWMPI_LIB}")
  endif()
  message(STATUS "FFTW headers: ${FFTW_INCLUDE}")
  add_definitions(-DHAVE_FFTW)
endif()

#---------------------- set include & link -----------------------------------#
# build executables
set (LIBDIRS numtools common boundary fluxes io mesh physics sources timedisc )

foreach (p ${LIBDIRS})
  include_directories(/.$(CMAKE_BINARY_DIR)/${p})
endforeach (p)
include_directories(/.$(CMAKE_BINARY_DIR))
include_directories(/.$(CMAKE_SOURCE_DIR)/common) #because of tap.h
if(FFTW_INCLUDE)
  include_directories(${FFTW_INCLUDE})
endif()
if(ASL_INCLUDE)
  include_directories(${ASL_INCLUDE})
endif()

foreach (p ${LIBDIRS})
  add_subdirectory (${p})
endforeach (p)

add_library(fosite fosite.f90)
target_link_libraries(fosite ${LIBDIRS})

# add gsl
if(GSL_FOUND)
  target_link_libraries(fosite ${GSL_LIBRARIES} ${FGSL_BINARY_DIR}/lib/libfgsl.a)
  add_dependencies(numtools fgsl)
endif()

# add fftw
if (NOT CMAKE_SYSTEM_NAME MATCHES "(.*)SX-Aurora")
  if(FFTW_LIB)
    target_link_libraries(fosite ${FFTW_LIB})
    if(PARALLEL)
      target_link_libraries(fosite ${FFTWMPI_LIB} m)
    endif()
  endif()
else()
  if(NOT PARALLEL)
    target_link_libraries(fosite ${FFTW_LIB} asl_sequential)
  else()
    target_link_libraries(fosite ${FFTWMPI_LIB} asl_mpi_sequential)
  endif()
endif()

add_subdirectory(examples)
add_subdirectory(tests)

#--------------------- doxygen documentation ----------------------------------#
option(BUILD_DOC "Build documentation" ON)

find_package(Doxygen 1.8.12)
if(DOXYGEN_FOUND)
  set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/doc/Doxyfile.in)
  set(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/doc/Doxyfile)

  # a quite clumpsy way to replace the placeholders in the doxygen document in
  # the right way but it was the fastest I found
  # 1. use glob (generates a list with ; as deliminter which doxygen cannot read)
  # 2. generate new custom string with every element in new line with \ at end
  FILE(GLOB htmllist ${CMAKE_CURRENT_SOURCE_DIR}/doc/html/*.html)
  foreach(line IN LISTS htmllist)
    string(APPEND htmlplaceholder "${line} \\ \n")
  endforeach()
  FILE(GLOB jslist ${CMAKE_CURRENT_SOURCE_DIR}/doc/html/js/*.js)
  foreach(line IN LISTS jslist)
    string(APPEND jsplaceholder "${line} \\ \n")
  endforeach()
  FILE(GLOB fontlist ${CMAKE_CURRENT_SOURCE_DIR}/doc/html/fonts/*)
  foreach(line IN LISTS fontlist)
    string(APPEND fontplaceholder "${line} \\ \n")
  endforeach()

  # copy doxygen file and replace all @strings@ within Doxygen.in
  configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)

  # add target "make doc"
  add_custom_target(doc
    # main command "doxygen Doxyfile" to run doxygen
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc
    COMMENT "Generating API documentation with Doxygen"

    COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/doc/adjust_names.sh ${CMAKE_CURRENT_BINARY_DIR}

    VERBATIM)
endif(DOXYGEN_FOUND)
