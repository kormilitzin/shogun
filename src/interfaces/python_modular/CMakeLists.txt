# find python library for the found python interpreter
# i.e. versions should match!
FIND_PACKAGE(PythonLibs ${PYTHON_VERSION_STRING} EXACT REQUIRED)
FIND_PACKAGE(NumPy REQUIRED)

SET(HAVE_PYTHON 1)

#custom swig flags for python modular interface
IF(${PYTHON_VERSION_MAJOR} VERSION_EQUAL 3)
  SET(TARGET_SWIGFLAGS "-builtin\;-modern\;-modernargs\;-threads\;-py3")
  SET(PYTHON3 1)
ELSE()
  SET(TARGET_SWIGFLAGS "-builtin\;-modern\;-modernargs\;-threads")
ENDIF()

# SWIG-generated Python-wrappers fail to build
# for Python >=3.5 with SWIG < 3.0.8.  Make CMake
# emit an error and fail to configure.
IF ((NOT "${PYTHON_VERSION_STRING}" VERSION_LESS "3.5") AND
  ("${SWIG_VERSION}" VERSION_LESS "3.0.8"))
  MESSAGE(FATAL_ERROR
    "Building Python_modular interface for Python >= 3.5 requires SWIG >= 3.0.8.")
ENDIF ((NOT "${PYTHON_VERSION_STRING}" VERSION_LESS "3.5") AND
  ("${SWIG_VERSION}" VERSION_LESS "3.0.8"))

include(CommonModularInterface)
include_directories(${PYTHON_INCLUDE_PATH} ${NUMPY_INCLUDE_DIRS})

#TODO: check for SWIG version where this bug has been applied already:
# https://github.com/swig/swig/pull/70

SET(SWIG_CXX_COMPILER_FLAGS "${SWIG_CXX_COMPILER_FLAGS} -Wno-c++11-narrowing")

GENERATE_MODULAR_TARGET(python ${CMAKE_CURRENT_SOURCE_DIR} ${PYTHON_LIBRARIES})

# pacakges directory
if(CMAKE_HOST_UNIX)
  execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from distutils.sysconfig import *; print(get_python_lib())"
                  RESULT_VARIABLE PYTHON_CVPY_PROCESS
                  OUTPUT_VARIABLE PYTHON_STD_PACKAGES_PATH
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  if("${PYTHON_STD_PACKAGES_PATH}" MATCHES "site-packages")
    set(_PYTHON_PACKAGES_PATH "python${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}/site-packages")
  else() #debian based assumed, install to the dist-packages.
    set(_PYTHON_PACKAGES_PATH "python${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}/dist-packages")
  endif()
  if(EXISTS "${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX}/${PYTHON_PACKAGES_PATH}")
    set(_PYTHON_PACKAGES_PATH "lib${LIB_SUFFIX}/${_PYTHON_PACKAGES_PATH}")
  else()
    set(_PYTHON_PACKAGES_PATH "lib/${_PYTHON_PACKAGES_PATH}")
  endif()
elseif(CMAKE_HOST_WIN32)
  get_filename_component(PYTHON_PATH "${PYTHON_EXECUTABLE}" PATH)
  file(TO_CMAKE_PATH "${PYTHON_PATH}" PYTHON_PATH)
  if(NOT EXISTS "${PYTHON_PATH}/Lib/site-packages")
    unset(PYTHON_PATH)
    get_filename_component(PYTHON_PATH "[HKEY_LOCAL_MACHINE\\SOFTWARE\\Python\\PythonCore\\${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}\\InstallPath]" ABSOLUTE)
    if(NOT PYTHON_PATH)
       get_filename_component(PYTHON_PATH "[HKEY_CURRENT_USER\\SOFTWARE\\Python\\PythonCore\\${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}\\InstallPath]" ABSOLUTE)
    endif()
    file(TO_CMAKE_PATH "${PYTHON_PATH}" PYTHON_PATH)
  endif()
  set(_PYTHON_PACKAGES_PATH "${PYTHON_PATH}/Lib/site-packages")
endif()
SET(PYTHON_PACKAGES_PATH "${_PYTHON_PACKAGES_PATH}" CACHE PATH "Where to install the python packages.")

INSTALL(TARGETS _python_modular
		DESTINATION ${PYTHON_PACKAGES_PATH}
    COMPONENT python)
INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/modshogun.py
		DESTINATION ${PYTHON_PACKAGES_PATH}
    COMPONENT python)

# create the __init.py__ files for modules and the main
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/__init__.py.in "from modshogun import *")
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/main_init.py.in "__all__= [\n")
configure_file(${CMAKE_CURRENT_BINARY_DIR}/__init__.py.in __init__.py COPYONLY)
SET(MODULES Kernel Distance Features Classifier Regression Converter
	Loss Clustering Evaluation IO Library Mathematics
	ModelSelection Preprocessor Structure Distribution Statistics Latent)

FOREACH(module ${MODULES})
	INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/__init__.py
		DESTINATION ${PYTHON_PACKAGES_PATH}/shogun/${module}
    COMPONENT python)
	file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/main_init.py.in "\"${module}\",\n")
ENDFOREACH()

file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/main_init.py.in "]")
configure_file(${CMAKE_CURRENT_BINARY_DIR}/main_init.py.in main_init.py COPYONLY)
INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/main_init.py
	DESTINATION ${PYTHON_PACKAGES_PATH}/shogun
  COMPONENT python
	RENAME __init__.py)
