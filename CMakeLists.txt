cmake_minimum_required (VERSION 3.10)

include (ExternalProject)

# Determine processor count
include(ProcessorCount)
ProcessorCount(Ncpu)


project (loongarch-lfs
    LANGUAGES C CXX
)


set (WORK_DIR "${CMAKE_CURRENT_BINARY_DIR}" CACHE FILEPATH "")
set (BUILD_DIR "${WORK_DIR}/build-lfs" CACHE FILEPATH "")
set (TARGET_SYSROOT "${WORK_DIR}/sysroot" CACHE FILEPATH "")
set (CROSS_TOOL_SRC_DIR "${PROJECT_SOURCE_DIR}/deps-crossbuild")
set (CROSS_TOOL_PREFIX "${WORK_DIR}/cross-tools")
set (DOWNLOAD_DIR "${PROJECT_SOURCE_DIR}/downloads")

set (CROSS_HOST x86_64-cross-linux-musl)
set (CROSS_TARGET loongarch64-unknown-linux-gnu)
set (MABI lp64)
set (BUILD_ARCH -march=loongarch)
set (BUILD_MABI -mabi=${MABI})
set (BUILD64 -mabi=lp64)
#export PATH=${SYSDIR}/cross-tools/bin:/bin:/usr/bin

option (CONFIG_PARALLEL_BUILD "Enable parallel build" ON)
if (CONFIG_PARALLEL_BUILD)
    set (MAKE_MULTIJOB -j${Ncpu})
endif ()

#
# Cross-build Tools
#

ExternalProject_Add (
    linux_src_downloader
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/linux_src

    GIT_REPOSITORY "git://github.com/loongson/linux.git" GIT_SHALLOW
    GIT_TAG origin/loongarch-next
    GIT_PROGRESS ON
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    BUILD_IN_SOURCE ON
    CONFIGURE_COMMAND make mrproper
    BUILD_COMMAND make ARCH=loongarch INSTALL_HDR_PATH=dest headers_install
    INSTALL_COMMAND ""
    BUILD_ALWAYS ON
)

ExternalProject_Add_Step (
    linux_src_downloader
    linux_header_install
    DEPENDEES build
    COMMAND find dest/include -name '.*' -delete
    COMMAND mkdir -pv ${TARGET_SYSROOT}/usr/include
    COMMAND cp -rv dest/include/ ${TARGET_SYSROOT}/usr/
    WORKING_DIRECTORY ${CROSS_TOOL_SRC_DIR}/linux_src
    ALWAYS ON
)

ExternalProject_Add (
    binutils_crossbuild
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/binutils_crossbuild

    GIT_REPOSITORY "git://github.com/loongson/binutils-gdb.git" GIT_SHALLOW
    GIT_TAG origin/loongarch-2_37
    GIT_PROGRESS ON
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    BUILD_IN_SOURCE ON      # Binutils will call for dependencies in source tree
    CONFIGURE_COMMAND CC=gcc AR=ar AS=as ${CROSS_TOOL_SRC_DIR}/binutils_crossbuild/configure --prefix=${CROSS_TOOL_PREFIX} --build=${CROSS_HOST} --host=${CROSS_HOST} --target=${CROSS_TARGET} --with-sysroot=${TARGET_SYSROOT} --disable-nls --disable-static --disable-werror --enable-64-bit-bfd
    BUILD_COMMAND make ${MAKE_MULTIJOB} configure-host && make ${MAKE_MULTIJOB}
    INSTALL_COMMAND make ${MAKE_MULTIJOB} install
)

ExternalProject_Add_Step (
    binutils_crossbuild
    binutils_crossbuild_prepatch
    DEPENDERS configure
    DEPENDEES download
    COMMAND rm -rf gdb libdecnumber readline sim
    WORKING_DIRECTORY ${CROSS_TOOL_SRC_DIR}/binutils_crossbuild
    ALWAYS OFF
)

ExternalProject_Add (
    gmp_crossbuild
    DOWNLOAD_DIR ${DOWNLOAD_DIR}
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/gmp_crossbuild

    URL "https://ftp.gnu.org/gnu/gmp/gmp-6.2.1.tar.xz"
    URL_MD5 0b82665c4a92fd2ade7440c13fcaa42b
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    CONFIGURE_COMMAND ${CROSS_TOOL_SRC_DIR}/gmp_crossbuild/configure --prefix=${CROSS_TOOL_PREFIX} --enable-cxx --disable-static
    BUILD_COMMAND make ${MAKE_MULTIJOB}
    INSTALL_COMMAND make ${MAKE_MULTIJOB} install
)

ExternalProject_Add (
    mpfr_crossbuild
    DOWNLOAD_DIR ${DOWNLOAD_DIR}
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/mpfr_crossbuild
    DEPENDS gmp_crossbuild

    URL "https://ftp.gnu.org/gnu/mpfr/mpfr-4.1.0.tar.xz"
    URL_MD5 bdd3d5efba9c17da8d83a35ec552baef
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    CONFIGURE_COMMAND ${CROSS_TOOL_SRC_DIR}/mpfr_crossbuild/configure --prefix=${CROSS_TOOL_PREFIX} --disable-static --with-gmp=${CROSS_TOOL_PREFIX}
    BUILD_COMMAND make ${MAKE_MULTIJOB}
    INSTALL_COMMAND make ${MAKE_MULTIJOB} install
)

ExternalProject_Add (
    mpc_crossbuild
    DOWNLOAD_DIR ${DOWNLOAD_DIR}
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/mpc_crossbuild
    DEPENDS gmp_crossbuild

    URL "https://ftp.gnu.org/gnu/mpc/mpc-1.2.1.tar.gz"
    URL_MD5 9f16c976c25bb0f76b50be749cd7a3a8
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    CONFIGURE_COMMAND ${CROSS_TOOL_SRC_DIR}/mpc_crossbuild/configure --prefix=${CROSS_TOOL_PREFIX} --disable-static --with-gmp=${CROSS_TOOL_PREFIX}
    BUILD_COMMAND make ${MAKE_MULTIJOB}
    INSTALL_COMMAND make ${MAKE_MULTIJOB} install
)

ExternalProject_Add (
    gcc_crossbuild
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/gcc_crossbuild
    DEPENDS binutils_crossbuild gmp_crossbuild mpfr_crossbuild mpc_crossbuild

    GIT_REPOSITORY "git://github.com/loongson/gcc.git" GIT_SHALLOW
    GIT_TAG origin/loongarch-12
    GIT_PROGRESS ON
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    BUILD_IN_SOURCE OFF
    CONFIGURE_COMMAND AR=ar LDFLAGS=-Wl,-rpath,${CROSS_TOOL_PREFIX}/lib ${CROSS_TOOL_SRC_DIR}/gcc_crossbuild/configure --prefix=${CROSS_TOOL_PREFIX} --build=${CROSS_HOST} --host=${CROSS_HOST} --target=${CROSS_TARGET} --disable-nls --with-mpfr=${CROSS_TOOL_PREFIX} --with-gmp=${CROSS_TOOL_PREFIX} --with-mpc=${CROSS_TOOL_PREFIX} --with-newlib --disable-shared --with-sysroot=${TARGET_SYSROOT} --disable-decimal-float --disable-libgomp --disable-libitm --disable-libsanitizer --disable-libquadmath --disable-threads --disable-target-zlib --with-system-zlib --enable-checking=release --with-abi=${MABI} --with-fix-loongson3-llsc --with-arch=loongarch --enable-languages=c --enable-tls

    BUILD_COMMAND make ${MAKE_MULTIJOB} all-gcc all-target-libgcc
    INSTALL_COMMAND make ${MAKE_MULTIJOB} install-gcc install-target-libgcc
)

set (GLIBC_ENV_ARG env PATH=${CROSS_TOOL_PREFIX}/bin:/bin:/usr/bin BUILD_CC=gcc CC=${CROSS_TARGET}-gcc\ ${BUILD64} CXX=${CROSS_TARGET}-gcc\ ${BUILD64} AR=${CROSS_TARGET}-ar RANLIB=${CROSS_TARGET}-ranlib)
ExternalProject_Add (
    glibc
    SOURCE_DIR ${CROSS_TOOL_SRC_DIR}/glibc
    DEPENDS binutils_crossbuild gcc_crossbuild linux_src_downloader

    GIT_REPOSITORY "git://github.com/loongson/glibc.git" GIT_SHALLOW
    GIT_TAG origin/loongarch_2_33
    GIT_PROGRESS ON
    UPDATE_DISCONNECTED true  # need this to avoid constant rebuild

    BUILD_IN_SOURCE OFF
    CONFIGURE_COMMAND ${GLIBC_ENV_ARG} ${CROSS_TOOL_SRC_DIR}/glibc/configure --prefix=/usr --host=${CROSS_TARGET} --build=${CROSS_HOST} --libdir=/usr/lib64 --libexecdir=/usr/lib64/glibc --enable-add-ons --with-tls --with-binutils=${CROSS_TOOL_PREFIX}/bin --with-headers=${TARGET_SYSROOT}/usr/include --enable-obsolete-rpc --disable-werror

    BUILD_COMMAND ${GLIBC_ENV_ARG} make ${MAKE_MULTIJOB}
    INSTALL_COMMAND make DESTDIR=${TARGET_SYSROOT} install
)
