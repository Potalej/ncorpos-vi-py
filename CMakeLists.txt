# ============================================================================
# Projeto
# ============================================================================
cmake_minimum_required(VERSION 3.15)

# Capturando a versao do pyproject.toml para centralizar
file(READ "${CMAKE_SOURCE_DIR}/pyproject.toml" PYPROJECT_CONTENTS)
string(REGEX MATCH "version *= *\"([^\"]+)\"" _ ${PYPROJECT_CONTENTS})
set(PACKAGE_VERSION "${CMAKE_MATCH_1}")

project(ncorpos_vi_py
  VERSION ${PACKAGE_VERSION}
  DESCRIPTION "Geracao de valores iniciais"
  LANGUAGES Fortran
)

option(SKBUILD "Should be ON of being build by skbuild, and OFF of being build by regular cmake" OFF)

if (NOT SKBUILD)
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/../cmake/")
endif()

# Safety net
if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
  message(
    FATAL_ERROR
      "In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there.\n"
  )
endif()

# ============================================================================
# Biblioteca Fortran (core)
# ============================================================================
message("\n> Dependencia: ncorpos-utilidades")
add_subdirectory("src/ncorpos_vi/ext/ncorpos_utilidades" utilidades)

message("\n> Dependencia: ncorpos-valores-iniciais")
add_subdirectory("src/ncorpos_vi/ext/ncorpos_valores_iniciais" valores_iniciais)

# Diretorio com os mods
set(MOD_DIR_UT ${utilidades_BINARY_DIR})
set(MOD_DIR_VI ${ncorpos_vi_BINARY_DIR})

# ============================================================================
# Python
# ============================================================================
find_package(Python3 REQUIRED COMPONENTS Interpreter)

# Gerar src/ncorpos_vi/_version.py
file(WRITE "${CMAKE_SOURCE_DIR}/src/ncorpos_vi/_version.py"
"__version__ = \"${PACKAGE_VERSION}\"\n")


# ============================================================================
# Wrapper f2py
# ============================================================================
set(F2PY_WRAPPER
  ${CMAKE_CURRENT_SOURCE_DIR}/src/ncorpos_vi/ext/ncorpos_vi.f90
)

set(F2PY_MODULE_NAME _core)

execute_process(
  COMMAND ${Python3_EXECUTABLE}
          -c
          "import sysconfig; print(sysconfig.get_config_var('EXT_SUFFIX'))"
  OUTPUT_VARIABLE PY_EXT_SUFFIX
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(F2PY_OUTPUT
  ${CMAKE_CURRENT_BINARY_DIR}/${F2PY_MODULE_NAME}${PY_EXT_SUFFIX}
)

add_custom_command(
  OUTPUT ${F2PY_OUTPUT}
  COMMAND
    ${Python3_EXECUTABLE} -m numpy.f2py
      -c
      -m ${F2PY_MODULE_NAME}
      ${F2PY_WRAPPER}

      --link-args=-Wl,-rpath,$ORIGIN
      --link-args=-lopenblas
      --link-args=-llapack

      -L$<TARGET_FILE_DIR:utilidades>
      -lutilidades
      -L$<TARGET_FILE_DIR:valores_iniciais>
      -lvalores_iniciais
      -I${MOD_DIR_UT}/mod
      -I${MOD_DIR_VI}/mod
      --quiet
  DEPENDS
    utilidades
    valores_iniciais
    ${F2PY_WRAPPER}
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "Gerando modulo Python via f2py com RPATH=$ORIGIN"
)

add_custom_target(python_module ALL
  DEPENDS ${F2PY_OUTPUT}
)

# ============================================================================
# Instala
# ============================================================================
message("\n> Instalacao")

# Modulo Python gerado pelo f2py
message(STATUS "Instalando _core")
install(
  FILES ${F2PY_OUTPUT}
  DESTINATION ncorpos_vi/_core
)

# Bibliotecas Fortran dependentes
message(STATUS "Instalando dependencias")
set_target_properties(utilidades PROPERTIES
  INSTALL_RPATH "$ORIGIN"
  BUILD_WITH_INSTALL_RPATH ON
)
set_target_properties(valores_iniciais PROPERTIES
  INSTALL_RPATH "$ORIGIN"
  BUILD_WITH_INSTALL_RPATH ON
)
install(
  TARGETS utilidades valores_iniciais
  LIBRARY DESTINATION ncorpos_vi/_core
)

# Arquivos Python do pacote
message(STATUS "Instalando pacote")
install(
  DIRECTORY ${CMAKE_SOURCE_DIR}/src/ncorpos_vi/
  DESTINATION ncorpos_vi
  FILES_MATCHING
    PATTERN "*.py"
)

message("")