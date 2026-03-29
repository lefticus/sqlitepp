cmake_minimum_required(VERSION 3.20)

file(READ "${CMAKE_CURRENT_SOURCE_DIR}/VERSION" SQLITE_VERSION)
string(STRIP "${SQLITE_VERSION}" SQLITE_VERSION)
project(sqlite3 VERSION ${SQLITE_VERSION} LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 11)

add_compile_options($<$<COMPILE_LANGUAGE:CXX>:-Wall> $<$<COMPILE_LANGUAGE:CXX>:-Werror>)

# --------------------------------------------------------------------------
# Options
# --------------------------------------------------------------------------
option(SQLITE_ENABLE_MATH_FUNCTIONS "Enable math SQL functions" ON)
option(SQLITE_ENABLE_FTS4 "Enable FTS4" ON)
option(SQLITE_ENABLE_FTS5 "Enable FTS5" OFF)
option(SQLITE_ENABLE_RTREE "Enable R-Tree" ON)
option(SQLITE_ENABLE_GEOPOLY "Enable Geopoly" OFF)
option(SQLITE_ENABLE_SESSION "Enable session extension" OFF)
option(SQLITE_ENABLE_PERCENTILE "Enable percentile function" ON)
option(SQLITE_ENABLE_DBPAGE_VTAB "Enable dbpage virtual table" ON)
option(SQLITE_ENABLE_DBSTAT_VTAB "Enable dbstat virtual table" ON)
option(SQLITE_ENABLE_BYTECODE_VTAB "Enable bytecode virtual table" ON)
option(SQLITE_ENABLE_STMTVTAB "Enable stmt virtual table" ON)
option(SQLITE_ENABLE_CARRAY "Enable carray table-valued function" ON)
option(SQLITE_ENABLE_COLUMN_METADATA "Enable column metadata" OFF)
option(SQLITE_ENABLE_EXPLAIN_COMMENTS "Enable explain comments" ON)
option(SQLITE_ENABLE_OFFSET_SQL_FUNC "Enable offset SQL function" ON)
option(SQLITE_ENABLE_UNKNOWN_SQL_FUNCTION "Enable unknown SQL function" ON)
option(SQLITE_ENABLE_MEMSYS5 "Enable memsys5 allocator" OFF)
option(SQLITE_ENABLE_MEMSYS3 "Enable memsys3 allocator" OFF)
option(SQLITE_ENABLE_SCANSTATUS "Enable scan status" OFF)
option(SQLITE_ENABLE_UPDATE_LIMIT "Enable UPDATE with LIMIT" OFF)
set(SQLITE_THREADSAFE "1" CACHE STRING "Thread safety mode (0=off, 1=serialized, 2=multi-thread)")
option(BUILD_SHARED_LIBS "Build shared library" ON)
option(BUILD_SHELL "Build sqlite3 CLI" ON)
option(BUILD_TESTING "Build test fixture" OFF)

# --------------------------------------------------------------------------
# Generate sqlite_cfg.h (replaces autosetup-generated config header)
# --------------------------------------------------------------------------
include(CheckIncludeFile)
include(CheckFunctionExists)
include(CheckTypeSize)
include(CheckLibraryExists)

check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(inttypes.h HAVE_INTTYPES_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(malloc.h HAVE_MALLOC_H)
check_include_file(dlfcn.h HAVE_DLFCN_H)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(dirent.h HAVE_DIRENT_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
check_include_file(sys/time.h HAVE_SYS_TIME_H)
check_include_file(zlib.h HAVE_ZLIB_H)

check_type_size(int8_t HAVE_INT8_T)
check_type_size(int16_t HAVE_INT16_T)
check_type_size(int32_t HAVE_INT32_T)
check_type_size(int64_t HAVE_INT64_T)
check_type_size(intptr_t HAVE_INTPTR_T)
check_type_size(uint8_t HAVE_UINT8_T)
check_type_size(uint16_t HAVE_UINT16_T)
check_type_size(uint32_t HAVE_UINT32_T)
check_type_size(uint64_t HAVE_UINT64_T)
check_type_size(uintptr_t HAVE_UINTPTR_T)

check_function_exists(fdatasync HAVE_FDATASYNC)
check_function_exists(gmtime_r HAVE_GMTIME_R)
check_function_exists(isnan HAVE_ISNAN)
check_function_exists(localtime_r HAVE_LOCALTIME_R)
check_function_exists(localtime_s HAVE_LOCALTIME_S)
check_function_exists(usleep HAVE_USLEEP)
check_function_exists(utime HAVE_UTIME)
check_function_exists(pread HAVE_PREAD)
check_function_exists(pread64 HAVE_PREAD64)
check_function_exists(pwrite HAVE_PWRITE)
check_function_exists(pwrite64 HAVE_PWRITE64)
check_function_exists(nanosleep HAVE_NANOSLEEP)
check_function_exists(realpath HAVE_REALPATH)
check_function_exists(dlopen HAVE_DLOPEN)
check_function_exists(ceil HAVE_CEIL_NOLM)

check_library_exists(m ceil "" HAVE_CEIL)
check_library_exists(z deflate "" HAVE_DEFLATE)
check_library_exists(pthread pthread_create "" HAVE_PTHREAD_CREATE)
check_library_exists(pthread pthread_mutexattr_init "" HAVE_PTHREAD_MUTEXATTR_INIT)

# Large file support
set(HAVE_LFS 1)

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake/sqlite_cfg.h.in"
  "${CMAKE_BINARY_DIR}/generated/sqlite_cfg.h"
)

# --------------------------------------------------------------------------
# Dependencies
# --------------------------------------------------------------------------
find_package(Threads)
find_library(MATH_LIBRARY m)
find_library(DL_LIBRARY dl)

find_package(ZLIB)
if(ZLIB_FOUND)
  set(SQLITE_HAVE_ZLIB 1)
endif()

# We need a TCL interpreter for code generation (jimsh built from source)
# and optionally tcl-dev for testfixture
if(BUILD_TESTING)
  find_path(TCL_INCLUDE_PATH tcl.h PATHS /usr/include/tcl8.6 /usr/include/tcl)
  find_library(TCL_LIBRARY NAMES tcl8.6 tcl)
  if(NOT TCL_INCLUDE_PATH OR NOT TCL_LIBRARY)
    message(FATAL_ERROR "TCL development files required for testing. Install tcl-dev.")
  endif()
  message(STATUS "TCL include: ${TCL_INCLUDE_PATH}")
  message(STATUS "TCL library: ${TCL_LIBRARY}")
endif()

# --------------------------------------------------------------------------
# Feature flag definitions
# --------------------------------------------------------------------------
set(SQLITE_FEATURE_DEFS "")

macro(sqlite_feature_flag OPTION_NAME DEFINE_NAME)
  if(${OPTION_NAME})
    list(APPEND SQLITE_FEATURE_DEFS "-D${DEFINE_NAME}")
  endif()
endmacro()

sqlite_feature_flag(SQLITE_ENABLE_MATH_FUNCTIONS SQLITE_ENABLE_MATH_FUNCTIONS)
sqlite_feature_flag(SQLITE_ENABLE_FTS4 SQLITE_ENABLE_FTS4)
sqlite_feature_flag(SQLITE_ENABLE_FTS5 SQLITE_ENABLE_FTS5)
sqlite_feature_flag(SQLITE_ENABLE_RTREE SQLITE_ENABLE_RTREE)
sqlite_feature_flag(SQLITE_ENABLE_GEOPOLY SQLITE_ENABLE_GEOPOLY)
sqlite_feature_flag(SQLITE_ENABLE_SESSION SQLITE_ENABLE_SESSION)
sqlite_feature_flag(SQLITE_ENABLE_PERCENTILE SQLITE_ENABLE_PERCENTILE)
sqlite_feature_flag(SQLITE_ENABLE_DBPAGE_VTAB SQLITE_ENABLE_DBPAGE_VTAB)
sqlite_feature_flag(SQLITE_ENABLE_DBSTAT_VTAB SQLITE_ENABLE_DBSTAT_VTAB)
sqlite_feature_flag(SQLITE_ENABLE_BYTECODE_VTAB SQLITE_ENABLE_BYTECODE_VTAB)
sqlite_feature_flag(SQLITE_ENABLE_STMTVTAB SQLITE_ENABLE_STMTVTAB)
sqlite_feature_flag(SQLITE_ENABLE_CARRAY SQLITE_ENABLE_CARRAY)
sqlite_feature_flag(SQLITE_ENABLE_COLUMN_METADATA SQLITE_ENABLE_COLUMN_METADATA)
sqlite_feature_flag(SQLITE_ENABLE_EXPLAIN_COMMENTS SQLITE_ENABLE_EXPLAIN_COMMENTS)
sqlite_feature_flag(SQLITE_ENABLE_OFFSET_SQL_FUNC SQLITE_ENABLE_OFFSET_SQL_FUNC)
sqlite_feature_flag(SQLITE_ENABLE_UNKNOWN_SQL_FUNCTION SQLITE_ENABLE_UNKNOWN_SQL_FUNCTION)
sqlite_feature_flag(SQLITE_ENABLE_MEMSYS5 SQLITE_ENABLE_MEMSYS5)
sqlite_feature_flag(SQLITE_ENABLE_MEMSYS3 SQLITE_ENABLE_MEMSYS3)
sqlite_feature_flag(SQLITE_ENABLE_SCANSTATUS SQLITE_ENABLE_SCANSTATUS)
sqlite_feature_flag(SQLITE_ENABLE_UPDATE_LIMIT SQLITE_ENABLE_UPDATE_LIMIT)

list(APPEND SQLITE_FEATURE_DEFS "-DSQLITE_THREADSAFE=${SQLITE_THREADSAFE}")

if(SQLITE_HAVE_ZLIB)
  list(APPEND SQLITE_FEATURE_DEFS "-DSQLITE_HAVE_ZLIB=1")
endif()

# --------------------------------------------------------------------------
# Build tools (compiled with the host C compiler)
# --------------------------------------------------------------------------
add_executable(jimsh autosetup/jimsh0.c)
target_compile_definitions(jimsh PRIVATE HAVE_REALPATH)
set_target_properties(jimsh PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/tools"
)

add_executable(lemon tool/lemon.c)
set_target_properties(lemon PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/tools"
)

add_executable(mkkeywordhash tool/mkkeywordhash.c)
target_compile_definitions(mkkeywordhash PRIVATE ${SQLITE_FEATURE_DEFS})
set_target_properties(mkkeywordhash PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/tools"
)

add_executable(mksourceid tool/mksourceid.c)
set_target_properties(mksourceid PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/tools"
)

set(JIMSH "$<TARGET_FILE:jimsh>")
set(LEMON "$<TARGET_FILE:lemon>")
set(MKKEYWORDHASH "$<TARGET_FILE:mkkeywordhash>")
set(MKSOURCEID "$<TARGET_FILE:mksourceid>")

set(TOP "${CMAKE_CURRENT_SOURCE_DIR}")
set(GEN "${CMAKE_BINARY_DIR}/generated")
file(MAKE_DIRECTORY "${GEN}")

# --------------------------------------------------------------------------
# Code generation: lempar.c (lemon template, must be in working dir)
# --------------------------------------------------------------------------
configure_file("${TOP}/tool/lempar.c" "${GEN}/lempar.c" COPYONLY)

# --------------------------------------------------------------------------
# Code generation: parse.c + parse.h (lemon parser)
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/parse.cpp" "${GEN}/parse.h"
  COMMAND ${CMAKE_COMMAND} -E copy "${TOP}/src/parse.y" "${GEN}/parse.y"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${LEMON} ${SQLITE_FEATURE_DEFS} -S "${GEN}/parse.y"
  COMMAND ${CMAKE_COMMAND} -E rename "${GEN}/parse.c" "${GEN}/parse.cpp"
  DEPENDS lemon "${TOP}/src/parse.y" "${GEN}/lempar.c"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating parser with lemon"
)

# --------------------------------------------------------------------------
# Code generation: opcodes.h (from parse.h + vdbe.cpp)
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/opcodes.h"
  COMMAND ${CMAKE_COMMAND} -E cat "${GEN}/parse.h" "${TOP}/src/vdbe.cpp"
          | ${JIMSH} "${TOP}/tool/mkopcodeh.tcl" > "${GEN}/opcodes.h"
  DEPENDS jimsh "${GEN}/parse.h" "${TOP}/src/vdbe.cpp"
          "${TOP}/tool/mkopcodeh.tcl"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating opcodes.h"
)

# --------------------------------------------------------------------------
# Code generation: opcodes.c
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/opcodes.cpp"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH} "${TOP}/tool/mkopcodec.tcl" "${GEN}/opcodes.h"
          > "${GEN}/opcodes.cpp"
  DEPENDS jimsh "${GEN}/opcodes.h" "${TOP}/tool/mkopcodec.tcl"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating opcodes.c"
)

# --------------------------------------------------------------------------
# Code generation: keywordhash.h
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/keywordhash.h"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${MKKEYWORDHASH} > "${GEN}/keywordhash.h"
  DEPENDS mkkeywordhash
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating keywordhash.h"
)

# --------------------------------------------------------------------------
# Code generation: pragma.h
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/pragma.h"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH} "${TOP}/tool/mkpragmatab.tcl" "${GEN}/pragma.h"
  DEPENDS jimsh "${TOP}/tool/mkpragmatab.tcl"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating pragma.h"
)

# --------------------------------------------------------------------------
# Code generation: ctime.c
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/ctime.cpp"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH} "${TOP}/tool/mkctimec.tcl" "${GEN}/ctime.c"
  COMMAND ${CMAKE_COMMAND} -E rename "${GEN}/ctime.c" "${GEN}/ctime.cpp"
  DEPENDS jimsh "${TOP}/tool/mkctimec.tcl"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating ctime.cpp"
)

# --------------------------------------------------------------------------
# Code generation: sqlite3.hpp (public header)
# --------------------------------------------------------------------------
add_custom_command(
  OUTPUT "${GEN}/sqlite3.hpp"
  COMMAND ${CMAKE_COMMAND} -E create_symlink ${MKSOURCEID} "${CMAKE_BINARY_DIR}/mksourceid"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH} "${TOP}/tool/mksqlite3h.tcl" "${TOP}"
          -o "${GEN}/sqlite3.hpp"
  DEPENDS jimsh mksourceid
          "${TOP}/src/sqlite.hpp.in"
          "${TOP}/ext/rtree/sqlite3rtree.hpp"
          "${TOP}/ext/session/sqlite3session.hpp"
          "${TOP}/ext/fts5/fts5.hpp"
          "${TOP}/manifest"
          "${TOP}/VERSION"
          "${TOP}/tool/mksqlite3h.tcl"
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
  COMMENT "Generating sqlite3.hpp"
)

# --------------------------------------------------------------------------
# Code generation: FTS5 (if enabled)
# --------------------------------------------------------------------------
# FTS5 amalgamation (always generate - testfixture needs it)
add_custom_command(
  OUTPUT "${GEN}/fts5parse.c" "${GEN}/fts5parse.h"
  COMMAND ${CMAKE_COMMAND} -E copy "${TOP}/ext/fts5/fts5parse.y" "${GEN}/fts5parse.y"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${LEMON} -S "${GEN}/fts5parse.y"
  DEPENDS lemon "${TOP}/ext/fts5/fts5parse.y" "${GEN}/lempar.c"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating FTS5 parser"
)
add_custom_command(
  OUTPUT "${GEN}/fts5.c"
  COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH}
          "${TOP}/ext/fts5/tool/mkfts5c.tcl"
  DEPENDS jimsh "${GEN}/fts5parse.c" "${GEN}/fts5parse.h"
  WORKING_DIRECTORY "${GEN}"
  COMMENT "Generating FTS5 amalgamation"
)
# Rename to .cpp for C++ compilation
add_custom_command(
  OUTPUT "${GEN}/fts5.cpp"
  COMMAND ${CMAKE_COMMAND} -E copy "${GEN}/fts5.c" "${GEN}/fts5.cpp"
  DEPENDS "${GEN}/fts5.c"
  COMMENT "Renaming fts5.c to fts5.cpp"
)

# --------------------------------------------------------------------------
# Custom target to ensure all generation runs
# --------------------------------------------------------------------------
add_custom_target(generated_sources DEPENDS
  "${GEN}/parse.cpp"
  "${GEN}/parse.h"
  "${GEN}/opcodes.h"
  "${GEN}/opcodes.cpp"
  "${GEN}/keywordhash.h"
  "${GEN}/pragma.h"
  "${GEN}/ctime.cpp"
  "${GEN}/sqlite3.hpp"
)

# --------------------------------------------------------------------------
# Library source files (non-amalgamation build)
# --------------------------------------------------------------------------
set(SQLITE_CORE_SOURCES
  src/alter.cpp
  src/analyze.cpp
  src/attach.cpp
  src/auth.cpp
  src/backup.cpp
  src/bitvec.cpp
  src/btmutex.cpp
  src/btree.cpp
  src/build.cpp
  src/callback.cpp
  src/carray.cpp
  src/complete.cpp
  src/date.cpp
  src/dbpage.cpp
  src/dbstat.cpp
  src/delete.cpp
  src/expr.cpp
  src/fault.cpp
  src/fkey.cpp
  src/func.cpp
  src/global.cpp
  src/hash.cpp
  src/insert.cpp
  src/json.cpp
  src/legacy.cpp
  src/loadext.cpp
  src/main.cpp
  src/malloc.cpp
  src/mem0.cpp
  src/mem1.cpp
  src/mem2.cpp
  src/mem3.cpp
  src/mem5.cpp
  src/memdb.cpp
  src/memjournal.cpp
  src/mutex.cpp
  src/mutex_noop.cpp
  src/mutex_unix.cpp
  src/mutex_w32.cpp
  src/notify.cpp
  src/os.cpp
  src/os_kv.cpp
  src/os_unix.cpp
  src/os_win.cpp
  src/pager.cpp
  src/pcache.cpp
  src/pcache1.cpp
  src/pragma.cpp
  src/prepare.cpp
  src/printf.cpp
  src/random.cpp
  src/resolve.cpp
  src/rowset.cpp
  src/select.cpp
  src/status.cpp
  src/table.cpp
  src/threads.cpp
  src/tokenize.cpp
  src/treeview.cpp
  src/trigger.cpp
  src/update.cpp
  src/upsert.cpp
  src/utf.cpp
  src/util.cpp
  src/vacuum.cpp
  src/vdbe.cpp
  src/vdbeapi.cpp
  src/vdbeaux.cpp
  src/vdbeblob.cpp
  src/vdbemem.cpp
  src/vdbesort.cpp
  src/vdbetrace.cpp
  src/vdbevtab.cpp
  src/vtab.cpp
  src/wal.cpp
  src/walker.cpp
  src/where.cpp
  src/wherecode.cpp
  src/whereexpr.cpp
  src/window.cpp
  # Generated sources
  "${GEN}/parse.cpp"
  "${GEN}/opcodes.cpp"
  "${GEN}/ctime.cpp"
  # Extensions compiled as part of core
  ext/fts3/fts3.cpp
  ext/fts3/fts3_aux.cpp
  ext/fts3/fts3_expr.cpp
  ext/fts3/fts3_hash.cpp
  ext/fts3/fts3_icu.cpp
  ext/fts3/fts3_porter.cpp
  ext/fts3/fts3_snippet.cpp
  ext/fts3/fts3_tokenizer.cpp
  ext/fts3/fts3_tokenizer1.cpp
  ext/fts3/fts3_tokenize_vtab.cpp
  ext/fts3/fts3_unicode.cpp
  ext/fts3/fts3_unicode2.cpp
  ext/fts3/fts3_write.cpp
  ext/rtree/rtree.cpp
  ext/session/sqlite3session.cpp
  ext/rbu/sqlite3rbu.cpp
  ext/misc/stmt.cpp
)

# --------------------------------------------------------------------------
# sqlite3 library
# --------------------------------------------------------------------------
add_library(sqlite3 ${SQLITE_CORE_SOURCES})
add_dependencies(sqlite3 generated_sources)

target_include_directories(sqlite3
  PUBLIC
    $<BUILD_INTERFACE:${GEN}>
    $<BUILD_INTERFACE:${TOP}/src>
    $<INSTALL_INTERFACE:include>
  PRIVATE
    ${GEN}
    ${TOP}/src
    ${TOP}/ext/rtree
    ${TOP}/ext/icu
    ${TOP}/ext/fts3
    ${TOP}/ext/session
    ${TOP}/ext/misc
)

target_compile_definitions(sqlite3
  PRIVATE
    ${SQLITE_FEATURE_DEFS}
    _HAVE_SQLITE_CONFIG_H
    BUILD_sqlite
    NDEBUG
    SQLITE_CORE
)

target_link_libraries(sqlite3
  PRIVATE
    $<$<BOOL:${Threads_FOUND}>:Threads::Threads>
    $<$<BOOL:${MATH_LIBRARY}>:${MATH_LIBRARY}>
    $<$<BOOL:${DL_LIBRARY}>:${DL_LIBRARY}>
    $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB>
)

set_target_properties(sqlite3 PROPERTIES
  VERSION ${PROJECT_VERSION}
  SOVERSION 0
  PUBLIC_HEADER "${GEN}/sqlite3.hpp;${TOP}/src/sqlite3ext.hpp"
)

# --------------------------------------------------------------------------
# sqlite3 CLI
# --------------------------------------------------------------------------
if(BUILD_SHELL)
  # Generate shell.cpp from shell.cpp.in
  add_custom_command(
    OUTPUT "${GEN}/shell.cpp"
    COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH} "${TOP}/tool/mkshellc.tcl" "${GEN}/shell.cpp"
    DEPENDS jimsh
            "${TOP}/src/shell.cpp.in"
            "${TOP}/tool/mkshellc.tcl"
    WORKING_DIRECTORY "${TOP}"
    COMMENT "Generating shell.cpp"
  )

  add_executable(sqlite3_shell "${GEN}/shell.cpp")
  add_dependencies(sqlite3_shell generated_sources)
  set_target_properties(sqlite3_shell PROPERTIES OUTPUT_NAME sqlite3)

  target_include_directories(sqlite3_shell PRIVATE ${GEN} ${TOP}/src)
  target_compile_definitions(sqlite3_shell PRIVATE
    ${SQLITE_FEATURE_DEFS}
    SQLITE_DQS=0
    SQLITE_ENABLE_BYTECODE_VTAB
    SQLITE_ENABLE_DBPAGE_VTAB
    SQLITE_ENABLE_DBSTAT_VTAB
    SQLITE_ENABLE_EXPLAIN_COMMENTS
    SQLITE_ENABLE_FTS4
    SQLITE_ENABLE_OFFSET_SQL_FUNC
    SQLITE_ENABLE_PERCENTILE
    SQLITE_ENABLE_RTREE
    SQLITE_ENABLE_STMTVTAB
    SQLITE_ENABLE_UNKNOWN_SQL_FUNCTION
    SQLITE_STRICT_SUBTYPE=1
    NDEBUG
  )

  target_link_libraries(sqlite3_shell PRIVATE
    sqlite3
    $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB>
  )
endif()

# --------------------------------------------------------------------------
# speedtest1
# --------------------------------------------------------------------------
add_executable(speedtest1 test/speedtest1.c)
set_source_files_properties(test/speedtest1.c PROPERTIES LANGUAGE CXX)
target_compile_options(speedtest1 PRIVATE -Wno-write-strings)
add_dependencies(speedtest1 generated_sources)
target_include_directories(speedtest1 PRIVATE ${GEN} ${TOP}/src)
target_compile_definitions(speedtest1 PRIVATE
  ${SQLITE_FEATURE_DEFS}
  SQLITE_OS_KV_OPTIONAL
  NDEBUG
)
target_link_libraries(speedtest1 PRIVATE sqlite3)

# --------------------------------------------------------------------------
# testfixture (TCL test harness)
# --------------------------------------------------------------------------
if(BUILD_TESTING)
  # tclsqlite-ex.c is generated by mkcombo.tcl
  add_custom_command(
    OUTPUT "${GEN}/tclsqlite-ex.cpp"
    COMMAND ${CMAKE_COMMAND} -E env TMPDIR=${CMAKE_BINARY_DIR} ${JIMSH}
            "${TOP}/tool/mkcombo.tcl"
            "${TOP}/ext/qrf/qrf.hpp" "${TOP}/ext/qrf/qrf.cpp"
            "${TOP}/src/tclsqlite.cpp"
            -o "${GEN}/tclsqlite-ex.cpp"
    DEPENDS jimsh
            "${TOP}/src/tclsqlite.cpp"
            "${TOP}/ext/qrf/qrf.cpp"
            "${TOP}/ext/qrf/qrf.hpp"
            "${TOP}/tool/mkcombo.tcl"
    WORKING_DIRECTORY "${TOP}"
    COMMENT "Generating tclsqlite-ex.cpp"
  )

  set(TESTSRC
    src/test1.cpp src/test2.cpp src/test3.cpp src/test4.cpp
    src/test5.cpp src/test6.cpp src/test8.cpp src/test9.cpp
    src/test_autoext.cpp src/test_backup.cpp src/test_bestindex.cpp
    src/test_blob.cpp src/test_btree.cpp src/test_config.cpp
    src/test_delete.cpp src/test_demovfs.cpp src/test_devsym.cpp
    src/test_fs.cpp src/test_func.cpp src/test_hexio.cpp
    src/test_init.cpp src/test_intarray.cpp src/test_journal.cpp
    src/test_malloc.cpp src/test_md5.cpp src/test_multiplex.cpp
    src/test_mutex.cpp src/test_onefile.cpp src/test_osinst.cpp
    src/test_pcache.cpp src/test_quota.cpp src/test_rtree.cpp
    src/test_schema.cpp src/test_superlock.cpp src/test_syscall.cpp
    src/test_tclsh.cpp src/test_tclvar.cpp src/test_thread.cpp
    src/test_vdbecov.cpp src/test_vfs.cpp src/test_window.cpp
    src/test_wsd.cpp
    ext/fts3/fts3_term.cpp ext/fts3/fts3_test.cpp
    ext/session/test_session.cpp
    ext/recover/test_recover.cpp
    ext/intck/sqlite3intck.cpp ext/intck/test_intck.cpp
    ext/rbu/test_rbu.cpp
    ext/expert/test_expert.cpp
    ext/fts5/fts5_tcl.cpp ext/fts5/fts5_test_mi.cpp ext/fts5/fts5_test_tok.cpp
    ext/misc/amatch.cpp ext/misc/appendvfs.cpp ext/misc/basexx.cpp
    ext/misc/cksumvfs.cpp ext/misc/closure.cpp ext/misc/csv.cpp
    ext/misc/decimal.cpp ext/misc/eval.cpp ext/misc/explain.cpp
    ext/misc/fileio.cpp ext/misc/fuzzer.cpp ext/misc/ieee754.cpp
    ext/misc/mmapwarm.cpp ext/misc/nextchar.cpp ext/misc/normalize.cpp
    ext/misc/prefixes.cpp ext/misc/qpvtab.cpp ext/misc/randomjson.cpp
    ext/misc/regexp.cpp ext/misc/remember.cpp ext/misc/series.cpp
    ext/misc/spellfix.cpp ext/misc/stmtrand.cpp ext/misc/totype.cpp
    ext/misc/unionvtab.cpp ext/misc/wholenumber.cpp ext/misc/zipfile.cpp
    ext/rtree/test_rtreedoc.cpp
    # Extensions needed by testfixture but not in SQLITE_CORE_SOURCES
    ext/expert/sqlite3expert.cpp
    ext/recover/sqlite3recover.cpp ext/recover/dbdata.cpp
    ext/intck/sqlite3intck.cpp
    # FTS5 amalgamation (generated)
    "${GEN}/fts5.cpp"
    "${GEN}/tclsqlite-ex.cpp"
  )

  add_executable(testfixture ${TESTSRC} ${SQLITE_CORE_SOURCES})
  add_dependencies(testfixture generated_sources)

  target_include_directories(testfixture PRIVATE
    ${GEN} ${TOP}/src
    ${TOP}/ext/rtree ${TOP}/ext/icu ${TOP}/ext/fts3
    ${TOP}/ext/session ${TOP}/ext/misc ${TOP}/ext/fts5
    ${TOP}/ext/recover ${TOP}/ext/expert ${TOP}/ext/intck
    ${TCL_INCLUDE_PATH}
  )

  target_compile_definitions(testfixture PRIVATE
    ${SQLITE_FEATURE_DEFS}
    _HAVE_SQLITE_CONFIG_H
    BUILD_sqlite
    SQLITE_CORE
    SQLITE_TEST=1
    SQLITE_CRASH_TEST=1
    SQLITE_NO_SYNC=1
    SQLITE_PRIVATE=extern
    TCLSH_INIT_PROC=sqlite3TestInit
    SQLITE_SERVER=1
    SQLITE_SERIES_CONSTRAINT_VERIFY=1
    SQLITE_DEFAULT_PAGE_SIZE=1024
    SQLITE_ENABLE_STMTVTAB
    SQLITE_ENABLE_DBPAGE_VTAB
    SQLITE_ENABLE_BYTECODE_VTAB
    SQLITE_ENABLE_CARRAY
    SQLITE_ENABLE_PERCENTILE
    SQLITE_CKSUMVFS_STATIC
    SQLITE_STATIC_RANDOMJSON
    SQLITE_STRICT_SUBTYPE=1
    SQLITE_ENABLE_FTS4
    SQLITE_ENABLE_FTS5
    SQLITE_ENABLE_RTREE
    SQLITE_ENABLE_GEOPOLY
    SQLITE_ENABLE_SESSION
    SQLITE_ENABLE_COLUMN_METADATA
    SQLITE_ENABLE_UPDATE_LIMIT
    SQLITE_ENABLE_MEMSYS5
    SQLITE_ENABLE_MEMSYS3
    SQLITE_ENABLE_SCANSTATUS
    SQLITE_ENABLE_MATH_FUNCTIONS
  )

  target_compile_options(testfixture PRIVATE -Wno-write-strings)

  target_link_libraries(testfixture PRIVATE
    ${TCL_LIBRARY}
    $<$<BOOL:${Threads_FOUND}>:Threads::Threads>
    $<$<BOOL:${MATH_LIBRARY}>:${MATH_LIBRARY}>
    $<$<BOOL:${DL_LIBRARY}>:${DL_LIBRARY}>
    $<$<BOOL:${ZLIB_FOUND}>:ZLIB::ZLIB>
  )
endif()

# --------------------------------------------------------------------------
# Install
# --------------------------------------------------------------------------
include(GNUInstallDirs)

install(TARGETS sqlite3
  EXPORT sqlite3Targets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

if(BUILD_SHELL)
  install(TARGETS sqlite3_shell
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  )
endif()

install(EXPORT sqlite3Targets
  FILE sqlite3Targets.cmake
  NAMESPACE SQLite3::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/sqlite3
)

# --------------------------------------------------------------------------
# pkg-config
# --------------------------------------------------------------------------
configure_file(
  "${TOP}/sqlite3.pc.in"
  "${CMAKE_BINARY_DIR}/sqlite3.pc"
  @ONLY
)
install(FILES "${CMAKE_BINARY_DIR}/sqlite3.pc"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)
