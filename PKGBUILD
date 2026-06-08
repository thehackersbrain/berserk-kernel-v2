# Maintainer: Gaurav Raj (@thehackersbrain) <gaurav@thehackersbrain.dev>

### BUILD OPTIONS

### Enable CachyOS config tuning (CONFIG_CACHY)
: "${_cachy_config:=yes}"

### Enable KBUILD_CFLAGS -O3
: "${_cc_harder:=yes}"

### Transparent Hugepages
# 'always' or 'madvise'
: "${_hugepage:=always}"

### Tick rate: 100, 250, 300, 500, 600, 750, 1000
: "${_HZ_ticks:=1000}"

### Tick type: periodic, idle, full
: "${_tickrate:=full}"

### Preempt type: full, lazy
: "${_preempt:=full}"

### Enable BBR3
: "${_tcp_bbr3:=yes}"

### Tweak kernel options prior to build via nconfig
: "${_makenconfig:=no}"

### Tweak kernel options prior to build via xconfig
: "${_makexconfig:=no}"

# ------------------------------------------------------------------------------

pkgbase=berserk-kernel
_major=7.0
_minor=11
_tagrel=0
pkgver=${_major}.${_minor}
pkgrel=1
_srcname=cachyos-${_major}.${_minor}-${_tagrel}
pkgdesc='The Berserk Kernel — CachyOS base with BORE + sched-ext and berserk tuning'
arch=(x86_64)
url='https://github.com/berserkarch/berserk-kernel'
license=(GPL-2.0-only)
makedepends=(
  bc
  binutils
  cpio
  gettext
  glibc
  libelf
  libgcc
  openssl
  pahole
  perl
  python
  rust
  rust-bindgen
  rust-src
  tar
  xxhash
  xz
  zlib
  zstd
)
options=('!strip' '!debug' '!lto')

_patchsource="https://raw.githubusercontent.com/cachyos/kernel-patches/master/${_major}"

source=(
  "https://github.com/CachyOS/linux/releases/download/${_srcname}/${_srcname}.tar.gz"
  "https://github.com/CachyOS/linux/releases/download/${_srcname}/${_srcname}.tar.gz.asc"
  "https://raw.githubusercontent.com/CachyOS/linux-cachyos/refs/heads/master/linux-cachyos-bore/config"
  "${_patchsource}/sched/0001-bore-cachy.patch"
)
validpgpkeys=(
  E18447AC260021D31F3FF6C4C8A2A4774B8B63C4 # Eric Naim <dnaim@cachyos.org>
  E8B9AA39F054E30E8290D492C3C4820857F654FE # Peter Jung <admin@ptr1337.dev>
)
sha256sums=('62502f9084899070f6939ff242fc6a8f676a3e3088d02ec6b653852a5741aa40'
  'SKIP'
  'ff6f8d9f1f3eb6f115e962cb3d6374d2c1e7a3f2449fc33cac62681a2440aa7e'
  'f594e3a0cf55649377e09bc22e6dd5152ecafe6a96460a68036a35bba5ba932e')

export KBUILD_BUILD_HOST=berserkarch
export KBUILD_BUILD_USER="$pkgbase"
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

_die() {
  error "$@"
  exit 1
}

prepare() {
  cd "$_srcname"

  echo "Setting version..."
  echo "-$pkgrel" >localversion.10-pkgrel
  echo "-berserk" >localversion.20-pkgname

  # Apply all patches from source[]
  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    src="${src%.zst}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 <"../$src"
  done

  # Apply berserk-specific patches on top if any exist
  if compgen -G "../patches/*.patch" >/dev/null 2>&1; then
    for p in ../patches/*.patch; do
      echo "Applying berserk patch $(basename "$p")..."
      patch -Np1 <"$p"
    done
  fi

  echo "Setting config..."
  cp ../config .config

  ### CPU optimization — native by default
  scripts/config -d GENERIC_CPU -d MZEN4 -e X86_NATIVE_CPU

  ### CachyOS config tuning
  if [ "$_cachy_config" = "yes" ]; then
    echo "Enabling CachyOS config (CONFIG_CACHY)..."
    scripts/config -e CACHY
  fi

  ### BORE scheduler + sched-ext framework for runtime hot-swap
  echo "Enabling BORE scheduler + sched-ext..."
  scripts/config -e SCHED_BORE -e SCHED_CLASS_EXT

  ### Tick rate
  case "$_HZ_ticks" in
  100 | 250 | 500 | 600 | 750 | 1000)
    scripts/config -d HZ_300 -e "HZ_${_HZ_ticks}" --set-val HZ "${_HZ_ticks}"
    ;;
  300)
    scripts/config -e HZ_300 --set-val HZ 300
    ;;
  *)
    _die "Invalid _HZ_ticks value: $_HZ_ticks"
    ;;
  esac
  echo "Tick rate: ${_HZ_ticks}Hz"

  ### Tick type
  case "$_tickrate" in
  periodic)
    scripts/config -d NO_HZ_IDLE -d NO_HZ_FULL -d NO_HZ -d NO_HZ_COMMON -e HZ_PERIODIC
    ;;
  idle)
    scripts/config -d HZ_PERIODIC -d NO_HZ_FULL -e NO_HZ_IDLE -e NO_HZ -e NO_HZ_COMMON
    ;;
  full)
    scripts/config -d HZ_PERIODIC -d NO_HZ_IDLE -d CONTEXT_TRACKING_FORCE \
      -e NO_HZ_FULL_NODEF -e NO_HZ_FULL -e NO_HZ -e NO_HZ_COMMON -e CONTEXT_TRACKING
    ;;
  *)
    _die "Invalid _tickrate value: $_tickrate"
    ;;
  esac
  echo "Tick type: $_tickrate"

  ### Preempt type
  case "$_preempt" in
  full) scripts/config -e PREEMPT -d PREEMPT_LAZY ;;
  lazy) scripts/config -d PREEMPT -e PREEMPT_LAZY ;;
  *) _die "Invalid _preempt value: $_preempt" ;;
  esac
  echo "Preempt: $_preempt"

  ### -O3
  if [ "$_cc_harder" = "yes" ]; then
    echo "Enabling -O3..."
    scripts/config -d CC_OPTIMIZE_FOR_PERFORMANCE \
      -e CC_OPTIMIZE_FOR_PERFORMANCE_O3
  fi

  ### BBR3
  if [ "$_tcp_bbr3" = "yes" ]; then
    echo "Enabling BBR3..."
    scripts/config \
      -m TCP_CONG_CUBIC \
      -d DEFAULT_CUBIC \
      -e TCP_CONG_BBR \
      -e DEFAULT_BBR \
      --set-str DEFAULT_TCP_CONG bbr \
      -m NET_SCH_FQ_CODEL \
      -e NET_SCH_FQ \
      -d CONFIG_DEFAULT_FQ_CODEL \
      -e CONFIG_DEFAULT_FQ
  fi

  ### Transparent Hugepages
  case "$_hugepage" in
  always) scripts/config -d TRANSPARENT_HUGEPAGE_MADVISE -e TRANSPARENT_HUGEPAGE_ALWAYS ;;
  madvise) scripts/config -d TRANSPARENT_HUGEPAGE_ALWAYS -e TRANSPARENT_HUGEPAGE_MADVISE ;;
  *) _die "Invalid _hugepage value: $_hugepage" ;;
  esac
  echo "THP: $_hugepage"

  ### Merge berserk config fragment on top if exists
  if [ -f ../berserk.config ]; then
    echo "Merging berserk.config fragment..."
    ./scripts/kconfig/merge_config.sh -m .config ../berserk.config
  fi

  echo "Rewriting configuration..."
  make olddefconfig
  diff -u ../config .config || :

  make -s kernelrelease >version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd "$_srcname"
  make -j"$(nproc)" all
  make -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    coreutils
    initramfs
    kmod
  )
  optdepends=(
    "$pkgbase-headers: headers and scripts for building modules"
    'linux-firmware: firmware images needed for some devices'
    'scx-scheds: to use sched-ext schedulers for runtime scheduler hot-swap'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(
    KSMBD-MODULE
    NTSYNC-MODULE
    VIRTUALBOX-GUEST-MODULES
    WIREGUARD-MODULE
  )

  cd "$_srcname"
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install

  rm "$modulesdir/build"
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(
    binutils
    glibc
    libelf
    libgcc
    openssl
    pahole
    xxhash
    zlib
    zstd
    "$pkgbase"
  )
  provides=(LINUX-HEADERS)

  cd "$_srcname"
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux tools/bpf/bpftool/vmlinux.h
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts
  ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"

  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  if [ -f tools/bpf/resolve_btfids/resolve_btfids ]; then
    install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids
  fi

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  if compgen -G "rust/*.rmeta" >/dev/null; then
    install -Dt "$builddir/rust" -m644 rust/*.rmeta
  fi
  if compgen -G "rust/*.so" >/dev/null; then
    install -Dt "$builddir/rust" rust/*.so
  fi

  echo "Installing unstripped VDSO..."
  make INSTALL_MOD_PATH="$pkgdir/usr" vdso_install link=

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
    application/x-sharedlib\;*) strip -v $STRIP_SHARED "$file" ;;
    application/x-archive\;*) strip -v $STRIP_STATIC "$file" ;;
    application/x-executable\;*) strip -v $STRIP_BINARIES "$file" ;;
    application/x-pie-executable\;*) strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
)
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
