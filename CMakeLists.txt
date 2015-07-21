cmake_minimum_required(VERSION 2.6)

project(libiconv)

set(PROJECT_VERSION 1.14)
set(PACKAGE_VERSION_MAJOR 1)
set(PACKAGE_VERSION_MINOR 14)
set(PACKAGE_VERSION_PATCH 0)
set(PACKAGE_VERSION_TWEAK 1)

option(BUILD_SHARED_LIBS "build a shared library" ON)

if(NOT DEFINED CMAKE_DEBUG_POSTFIX)
	set(CMAKE_DEBUG_POSTFIX "d")
endif()

set(CMAKE_C_VISIBILITY_PRESET hidden)

if(CMAKE_COMPILER_IS_GNUCC)
	include(CheckCCompilerFlag)
	check_c_compiler_flag(-fvisibility=hidden HAVE_VISIBILITY)
endif()

unset(CONFIGURED_DEFINES)
macro(set_define name value)
	set(${name} ${value})
	list(APPEND CONFIGURED_DEFINES ${name})
endmacro()

if(HAVE_VISIBILITY)
	set_define(HAVE_VISIBILITY 1)
else()
	set_define(HAVE_VISIBILITY 0)
endif()
set_define(ICONV_CONST const)
set_define(USE_MBSTATE_T 0)
set_define(BROKEN_WCHAR_H 0)
set_define(HAVE_WCHAR_T 1)

add_definitions(
	-DBUILDING_LIBICONV
	-DBUILDING_LIBCHARSET
	-DPACKAGE_VERSION_STRING=${PROJECT_VERSION}
	-DPACKAGE_VERSION_MAJOR=${PACKAGE_VERSION_MAJOR}
	-DPACKAGE_VERSION_MINOR=${PACKAGE_VERSION_MINOR}
	-DPACKAGE_VERSION_SUBMINOR=${PACKAGE_VERSION_PATCH}
	-DLIBDIR
)
if(NOT BUILD_SHARED_LIBS)
	add_definitions(-DLIBICONV_STATIC -DLIBCHARSET_STATIC)
endif()


### Configure config.h
file(READ lib/config.h.in CONFIG_H_CONTENT)
string(REGEX REPLACE "\n#undef" "\n//#undef" CONFIG_H_CONTENT "${CONFIG_H_CONTENT}")
foreach(name ${CONFIGURED_DEFINES})
	string(REPLACE "//#undef ${name}" "#define ${name} ${${name}}" CONFIG_H_CONTENT "${CONFIG_H_CONTENT}")
endforeach()
string(REPLACE "#define ICONV_CONST /* empty by default */" "#define ICONV_CONST ${ICONV_CONST}" CONFIG_H_CONTENT "${CONFIG_H_CONTENT}")
string(CONFIGURE "${CONFIG_H_CONTENT}" CONFIG_H_CONTENT @ONLY)
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/config.h" "${CONFIG_H_CONTENT}")


### Configure iconv.h
file(READ include/iconv.h.in ICONV_H_CONTENT)
string(REPLACE "#define _LIBICONV_H" "#define _LIBICONV_H

#ifdef _MSC_VER
#	ifndef LIBICONV_STATIC
#		ifdef BUILDING_LIBICONV
#			define LIBICONV_API __declspec(dllexport)
#		else
#			define LIBICONV_API __declspec(dllimport)
#		endif
#	else
#		define LIBICONV_API
#	endif
#else
#	if ${HAVE_VISIBILITY} && BUILDING_LIBICONV
#		define LIBICONV_API __attribute__((__visibility__(\"default\")))
#	else
#		define LIBICONV_API
#	endif
#endif //_MSC_VER" ICONV_H_CONTENT "${ICONV_H_CONTENT}")
string(REGEX REPLACE "\n[ \t]*extern " "\nextern LIBICONV_API " ICONV_H_CONTENT "${ICONV_H_CONTENT}")
string(REPLACE "extern LIBICONV_API \"C\"" "extern \"C\"" ICONV_H_CONTENT "${ICONV_H_CONTENT}")
foreach(name ${CONFIGURED_DEFINES})
	string(REPLACE "//#undef ${name}" "#define ${name} ${${name}}" ICONV_H_CONTENT "${ICONV_H_CONTENT}")
endforeach()
string(CONFIGURE "${ICONV_H_CONTENT}" ICONV_H_CONTENT @ONLY)
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/iconv.h" "${ICONV_H_CONTENT}")


### Configure localcharset.h
file(READ libcharset/include/localcharset.h.in LOCALCHARSET_H_CONTENT)
string(REPLACE "#define _LOCALCHARSET_H" "#define _LOCALCHARSET_H

#ifdef _MSC_VER
#	ifndef LIBCHARSET_STATIC
#		ifdef BUILDING_LIBCHARSET
#			define LIBCHARSET_API __declspec(dllexport)
#		else
#			define LIBCHARSET_API __declspec(dllimport)
#		endif
#	else
#		define LIBCHARSET_API
#	endif
#else
#	if ${HAVE_VISIBILITY} && BUILDING_LIBCHARSET
#		define LIBCHARSET_API __attribute__((__visibility__(\"default\")))
#	else
#		define LIBCHARSET_API
#	endif
#endif //_MSC_VER" LOCALCHARSET_H_CONTENT "${LOCALCHARSET_H_CONTENT}")
string(REGEX REPLACE "\n[ \t]*extern " "\nextern LIBCHARSET_API " LOCALCHARSET_H_CONTENT "${LOCALCHARSET_H_CONTENT}")
string(REPLACE "extern LIBCHARSET_API \"C\"" "extern \"C\"" LOCALCHARSET_H_CONTENT "${LOCALCHARSET_H_CONTENT}")
foreach(name ${CONFIGURED_DEFINES})
	string(REPLACE "//#undef ${name}" "#define ${name} ${${name}}" LOCALCHARSET_H_CONTENT "${LOCALCHARSET_H_CONTENT}")
endforeach()
string(CONFIGURE "${LOCALCHARSET_H_CONTENT}" LOCALCHARSET_H_CONTENT @ONLY)
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/localcharset.h" "${LOCALCHARSET_H_CONTENT}")


# TODO: Make this work on Windows, too
#add_executable(genaliases lib/genaliases.c)


# TODO: Make this work on Windows, too
#add_executable(genaliases2 lib/genaliases2.c)


### Generate flags.h
add_executable(genflags lib/genflags.c)
add_custom_command(
	OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/flags.h"
	COMMAND genflags > "${CMAKE_CURRENT_BINARY_DIR}/flags.h"
	VERBATIM
)


# TODO: Make this work on Windows, too
#add_executable(gentranslit lib/gentranslit.c)


### libiconv
set(LIBICONV_SOURCES
	libcharset/lib/localcharset.c
	lib/relocatable.c
	lib/iconv.c
	windows/libiconv.rc
)

set(LIBICONV_PRIVATE_HEADERS
	"${CMAKE_CURRENT_BINARY_DIR}/config.h"
	"${CMAKE_CURRENT_BINARY_DIR}/localcharset.h"
	"${CMAKE_CURRENT_BINARY_DIR}/flags.h"
	lib/aliases.h
	lib/aliases2.h
	lib/aliases_aix.h
	lib/aliases_aix_sysaix.h
	lib/aliases_dos.h
	lib/aliases_extra.h
	lib/aliases_osf1.h
	lib/aliases_osf1_sysosf1.h
	lib/aliases_sysaix.h
	lib/aliases_syshpux.h
	lib/aliases_sysosf1.h
	lib/aliases_syssolaris.h
	lib/armscii_8.h
	lib/ascii.h
	lib/atarist.h
	lib/big5.h
	lib/big5hkscs1999.h
	lib/big5hkscs2001.h
	lib/big5hkscs2004.h
	lib/big5hkscs2008.h
	lib/big5_2003.h
	lib/c99.h
	lib/canonical.h
	lib/canonical_aix.h
	lib/canonical_aix_sysaix.h
	lib/canonical_dos.h
	lib/canonical_extra.h
	lib/canonical_local.h
	lib/canonical_local_sysaix.h
	lib/canonical_local_syshpux.h
	lib/canonical_local_sysosf1.h
	lib/canonical_local_syssolaris.h
	lib/canonical_osf1.h
	lib/canonical_osf1_sysosf1.h
	lib/canonical_sysaix.h
	lib/canonical_syshpux.h
	lib/canonical_sysosf1.h
	lib/canonical_syssolaris.h
	lib/ces_big5.h
	lib/ces_gbk.h
	lib/cjk_variants.h
	lib/cns11643.h
	lib/cns11643_1.h
	lib/cns11643_15.h
	lib/cns11643_2.h
	lib/cns11643_3.h
	lib/cns11643_4.h
	lib/cns11643_4a.h
	lib/cns11643_4b.h
	lib/cns11643_5.h
	lib/cns11643_6.h
	lib/cns11643_7.h
	lib/cns11643_inv.h
	lib/converters.h
	lib/cp1046.h
	lib/cp1124.h
	lib/cp1125.h
	lib/cp1129.h
	lib/cp1131.h
	lib/cp1133.h
	lib/cp1161.h
	lib/cp1162.h
	lib/cp1163.h
	lib/cp1250.h
	lib/cp1251.h
	lib/cp1252.h
	lib/cp1253.h
	lib/cp1254.h
	lib/cp1255.h
	lib/cp1256.h
	lib/cp1257.h
	lib/cp1258.h
	lib/cp437.h
	lib/cp50221_0208_ext.h
	lib/cp50221_0212_ext.h
	lib/cp737.h
	lib/cp775.h
	lib/cp850.h
	lib/cp852.h
	lib/cp853.h
	lib/cp855.h
	lib/cp856.h
	lib/cp857.h
	lib/cp858.h
	lib/cp860.h
	lib/cp861.h
	lib/cp862.h
	lib/cp863.h
	lib/cp864.h
	lib/cp865.h
	lib/cp866.h
	lib/cp869.h
	lib/cp874.h
	lib/cp922.h
	lib/cp932.h
	lib/cp932ext.h
	lib/cp936.h
	lib/cp936ext.h
	lib/cp943.h
	lib/cp949.h
	lib/cp950.h
	lib/cp950ext.h
	lib/dec_hanyu.h
	lib/dec_kanji.h
	lib/euc_cn.h
	lib/euc_jisx0213.h
	lib/euc_jp.h
	lib/euc_kr.h
	lib/euc_tw.h
	lib/flushwc.h
	lib/gb12345.h
	lib/gb12345ext.h
	lib/gb18030.h
	lib/gb18030ext.h
	lib/gb18030uni.h
	lib/gb2312.h
	lib/gbk.h
	lib/gbkext1.h
	lib/gbkext2.h
	lib/gbkext_inv.h
	lib/georgian_academy.h
	lib/georgian_ps.h
	lib/hkscs1999.h
	lib/hkscs2001.h
	lib/hkscs2004.h
	lib/hkscs2008.h
	lib/hp_roman8.h
	lib/hz.h
	lib/iconv_open1.h
	lib/iconv_open2.h
	lib/iso2022_cn.h
	lib/iso2022_cnext.h
	lib/iso2022_jp.h
	lib/iso2022_jp1.h
	lib/iso2022_jp2.h
	lib/iso2022_jp3.h
	lib/iso2022_jpms.h
	lib/iso2022_kr.h
	lib/iso646_cn.h
	lib/iso646_jp.h
	lib/iso8859_1.h
	lib/iso8859_10.h
	lib/iso8859_11.h
	lib/iso8859_13.h
	lib/iso8859_14.h
	lib/iso8859_15.h
	lib/iso8859_16.h
	lib/iso8859_2.h
	lib/iso8859_3.h
	lib/iso8859_4.h
	lib/iso8859_5.h
	lib/iso8859_6.h
	lib/iso8859_7.h
	lib/iso8859_8.h
	lib/iso8859_9.h
	lib/isoir165.h
	lib/isoir165ext.h
	lib/java.h
	lib/jisx0201.h
	lib/jisx0208.h
	lib/jisx0212.h
	lib/jisx0213.h
	lib/johab.h
	lib/johab_hangul.h
	lib/koi8_r.h
	lib/koi8_ru.h
	lib/koi8_t.h
	lib/koi8_u.h
	lib/ksc5601.h
	lib/loops.h
	lib/loop_unicode.h
	lib/loop_wchar.h
	lib/mac_arabic.h
	lib/mac_centraleurope.h
	lib/mac_croatian.h
	lib/mac_cyrillic.h
	lib/mac_greek.h
	lib/mac_hebrew.h
	lib/mac_iceland.h
	lib/mac_roman.h
	lib/mac_romania.h
	lib/mac_thai.h
	lib/mac_turkish.h
	lib/mac_ukraine.h
	lib/mulelao.h
	lib/nextstep.h
	lib/pt154.h
	lib/relocatable.h
	lib/riscos1.h
	lib/rk1048.h
	lib/shift_jisx0213.h
	lib/sjis.h
	lib/tcvn.h
	lib/tds565.h
	lib/tis620.h
	lib/translit.h
	lib/ucs2.h
	lib/ucs2be.h
	lib/ucs2internal.h
	lib/ucs2le.h
	lib/ucs2swapped.h
	lib/ucs4.h
	lib/ucs4be.h
	lib/ucs4internal.h
	lib/ucs4le.h
	lib/ucs4swapped.h
	lib/uhc_1.h
	lib/uhc_2.h
	lib/utf16.h
	lib/utf16be.h
	lib/utf16le.h
	lib/utf32.h
	lib/utf32be.h
	lib/utf32le.h
	lib/utf7.h
	lib/utf8.h
	lib/vietcomb.h
	lib/viscii.h
	lib/encodings.def
	lib/encodings_aix.def
	lib/encodings_dos.def
	lib/encodings_extra.def
	lib/encodings_local.def
	lib/encodings_osf1.def
	lib/translit.def
)

set(LIBICONV_PUBLIC_HEADERS
	"${CMAKE_CURRENT_BINARY_DIR}/iconv.h"
)

include_directories(include "${CMAKE_CURRENT_BINARY_DIR}" lib)

add_library(iconv ${LIBICONV_SOURCES} ${LIBICONV_PRIVATE_HEADERS} ${LIBICONV_PUBLIC_HEADERS})

install(TARGETS iconv
	ARCHIVE DESTINATION lib
	LIBRARY DESTINATION lib
	RUNTIME DESTINATION bin
)

if(MSVC)
	if(BUILD_SHARED_LIBS)
		install(FILES "$<TARGET_PDB_FILE:iconv>" DESTINATION bin)
	endif()
endif()

install(FILES ${LIBICONV_PUBLIC_HEADERS} DESTINATION include)
