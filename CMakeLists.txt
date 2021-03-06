cmake_minimum_required(VERSION 2.6)
project(ecell4-notebooks)
set(PROJECT_VERSION 4.1.0)

find_package(PythonInterp ${PY_VERSION})
if (NOT PYTHONINTERP_FOUND)
    message(FATAL_ERROR "No Python interpreter was found. Set PYTHON_EXECUTABLE:FILEPATH.")
endif()

if (DEFINED PYTHON_VERSION_MAJOR AND DEFINED PYTHON_VERSION_MINOR)
    set(PYTHON_VERSION ${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR})
else (DEFINED PYTHON_VERSION_MAJOR AND DEFINED PYTHON_VERSION_MINOR)
    execute_process(COMMAND ${PYTHON_EXECUTABLE} --version
        OUTPUT_VARIABLE PYTHON_VERSION_RAW
        ERROR_VARIABLE PYTHON_VERSION_RAW)
    string(REPLACE "\n" "" PYTHON_VERSION_RAW "${PYTHON_VERSION_RAW}")
    string(REGEX MATCH "([0-9]+)\\.([0-9]+)" PYTHON_VERSION "${PYTHON_VERSION_RAW}")
    set(PYTHON_VERSION_MAJOR ${CMAKE_MATCH_1})
    set(PYTHON_VERSION_MINOR ${CMAKE_MATCH_2})
endif (DEFINED PYTHON_VERSION_MAJOR AND DEFINED PYTHON_VERSION_MINOR)
# message("${PYTHON_VERSION} ${PYTHON_VERSION_MAJOR} ${PYTHON_VERSION_MINOR}")

find_program(JUPYTER_EXECUTABLE jupyter)

message("JUPYTER_EXECUTABLE=${JUPYTER_EXECUTABLE}")

if (NOT IPYTHON_KERNEL_NAME)
  set(IPYTHON_KERNEL_NAME "python${PYTHON_VERSION_MAJOR}")
endif (NOT IPYTHON_KERNEL_NAME)

message("IPYTHON_KERNEL_NAME=${IPYTHON_KERNEL_NAME}")

add_custom_target(build_directories ALL
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/temp
  COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_BINARY_DIR}/release)

function(execute_notebooks out_var)
  set(result)
  foreach(in_f ${ARGN})
    get_filename_component(in_dir ${in_f} DIRECTORY)
    get_filename_component(in_name ${in_f} NAME)
    file(RELATIVE_PATH rel_path ${CMAKE_CURRENT_SOURCE_DIR} ${in_dir})
    set(tmp_dir "${CMAKE_CURRENT_BINARY_DIR}/temp/ipynb/${rel_path}")
    set(out_dir "${CMAKE_CURRENT_BINARY_DIR}/release/ipynb/${rel_path}")
    set(out_f "${out_dir}/${in_name}")

    add_custom_command(
      OUTPUT ${out_f}
      COMMAND ${JUPYTER_EXECUTABLE} nbconvert --to notebook --nbformat 4 --output-dir=${tmp_dir} ${in_f}
      COMMAND ${JUPYTER_EXECUTABLE} nbconvert --to notebook --execute --output-dir=${out_dir} --ExecutePreprocessor.timeout=18000 --ExecutePreprocessor.kernel_name='${IPYTHON_KERNEL_NAME}' ${tmp_dir}/${in_name}
      DEPENDS ${in_f} build_directories
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
      COMMENT "Executing a notebook ... [${out_f}]"
      VERBATIM
      )
    list(APPEND result ${out_f})
  endforeach()
  set(${out_var} "${result}" PARENT_SCOPE)
endfunction()

function(convert_notebooks out_var ext_name)
  set(result)
  foreach(in_f ${ARGN})
    get_filename_component(in_dir ${in_f} DIRECTORY)
    get_filename_component(in_name ${in_f} NAME)
    file(RELATIVE_PATH rel_path ${CMAKE_CURRENT_SOURCE_DIR} ${in_dir})
    set(tmp_dir "${CMAKE_CURRENT_BINARY_DIR}/temp/${ext_name}/${rel_path}")
    set(ipynb_dir "${CMAKE_CURRENT_BINARY_DIR}/release/ipynb/${rel_path}")
    set(out_dir "${CMAKE_CURRENT_BINARY_DIR}/release/${ext_name}/${rel_path}")

    string(REGEX REPLACE "\\.[^.]*$" "" out_f ${in_name})
    set(out_d "${out_dir}/${out_f}_files")
    set(out_f "${out_dir}/${out_f}.${ext_name}")

    add_custom_command(
      OUTPUT ${out_f} ${out_d}
      COMMAND ${JUPYTER_EXECUTABLE} nbconvert --to ${ext_name} --output-dir=${out_dir} ${ipynb_dir}/${in_name}
      DEPENDS ${in_f} ${ipynb_dir}/${in_name}
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
      COMMENT "Translating a notebook in ${ext_name} ... [${out_f}, ${out_d}]"
      VERBATIM
      )
    list(APPEND result ${out_f})
    list(APPEND retult ${out_d})
  endforeach()
  set(${out_var} "${result}" PARENT_SCOPE)
endfunction()

# file(GLOB IPYNB_FILES en/index.ipynb en/tutorial*.ipynb en/example*.ipynb en/time-consuming/example*.ipynb Tests/*.ipynb)
file(GLOB IPYNB_FILES
    en/index.ipynb
    en/tutorials/tutorial[1-4].ipynb en/tutorials/tutorial[8-9].ipynb en/tutorials/tutorial10.ipynb
    en/examples/example[1-6].ipynb en/examples/example1[0-1].ipynb)

execute_notebooks(notebooks ${IPYNB_FILES})
convert_notebooks(rsts rst ${IPYNB_FILES})
convert_notebooks(htmls html ${IPYNB_FILES})

add_custom_target(notebooks ALL DEPENDS ${notebooks})
add_custom_target(rsts ALL DEPENDS ${rsts})
add_custom_target(htmls ALL DEPENDS ${htmls})

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}//release/ DESTINATION .)
