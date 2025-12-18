# ============================================================================
# Projeto
# ============================================================================
cmake_minimum_required(VERSION 3.15)

project(ncorpos_vi
  VERSION 0.1.0
  DESCRIPTION "Geracao de valores iniciais"
  LANGUAGES Fortran
)

enable_language(Fortran)

include(FetchContent)
include(repos.cmake)
find_package(Python3 REQUIRED COMPONENTS Interpreter Development)

# ============================================================================
# Biblioteca Fortran (core)
# ============================================================================
message(STATUS "Baixando a biblioteca 'ncorpos_vi'...")
FetchContent_Declare(
  ncorpos_vi_f
  GIT_REPOSITORY ${VALORES_INICIAIS_REPO}
  GIT_TAG ${VALORES_INICIAIS_TAG}
)
FetchContent_MakeAvailable(ncorpos_vi_f)

# Diretorio com os mods
message("${ncorpos_vi_f_BINARY_DIR}")
set(MOD_DIR ${ncorpos_vi_f_BINARY_DIR})

# ============================================================================
# Wrapper f2py
# ============================================================================
set(F2PY_WRAPPER
  ${CMAKE_CURRENT_SOURCE_DIR}/bindings/ncorpos_vi.f90
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

MESSAGE("suspeito: ${CMAKE_CURRENT_BINARY_DIR}")

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

      -L$<TARGET_FILE_DIR:valores_iniciais>
      -lvalores_iniciais
      -L$<TARGET_FILE_DIR:utilidades>
      -lutilidades
      -I${MOD_DIR}/mod
      --quiet
  DEPENDS
    valores_iniciais
    utilidades
    ${F2PY_WRAPPER}
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "Gerando modulo Python via f2py com RPATH=$ORIGIN"
)


add_custom_target(python_module ALL
  DEPENDS ${F2PY_OUTPUT}
)

# ============================================================================
# INSTALL — ISSO É O QUE FALTAVA
# ============================================================================

# Módulo Python gerado pelo f2py
install(
  FILES ${F2PY_OUTPUT}
  DESTINATION ncorpos_vi/_core
)

# Bibliotecas Fortran dependentes
install(
  TARGETS valores_iniciais utilidades
  LIBRARY DESTINATION ncorpos_vi/_core
)

# Arquivos Python do pacote
install(
  DIRECTORY ${CMAKE_SOURCE_DIR}/src/ncorpos_vi/
  DESTINATION ncorpos_vi
  FILES_MATCHING
    PATTERN "*.py"
)