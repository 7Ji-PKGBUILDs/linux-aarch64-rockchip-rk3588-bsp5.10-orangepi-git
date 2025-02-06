# Maintainer: 7Ji <pugokughin@gmail.com>

_desc="AArch64 vendor kernel for Orange Pi 5/5B/5Plus/5Pro (git version)"
_orangepi_repo=linux-orangepi
_srcname=${_orangepi_repo}
_pkgbase=linux-aarch64-rockchip-rk3588-bsp5.10-orangepi
pkgbase=${_pkgbase}-git
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
pkgver=5.10.160.r1080214.4b357fedbdeb.f479be9
pkgrel=1
arch=(aarch64)
_gh_ornagepi=https://github.com/orangepi-xunlong
url="${_gh_ornagepi}/${_orangepi_repo}"
license=(GPL2)
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools' 'git' 'python'
)
options=(!strip !distcc)
source=(
  "git+${url}.git#branch=orange-pi-5.10-rk35xx"
  "git+${_gh_ornagepi}/orangepi-build.git#branch=next"
  '0001-rga3_uncompact_fix.patch'
  '0002-vop2_rgba2101010_capability_fix.patch'
  '0003-tp_link_ub500.patch'
)
sha256sums=(
  'SKIP'
  'SKIP'
  '5895a048c1074336eda07f702f76386a7cf7312c2d53bb5e179171c61c420ed7'
  'e4b889179584493256b45acda451c4d7f0d15a4d9d4e7731c9577ac5b7096adc'
  '99dfd35ad2ed47b8d838d689844646c62aba136b4c62ec5e36fe9556c2d504bc'
)

_config=external/config/kernel/linux-rockchip-rk3588-legacy.config

prepare() {
  echo "Setting version..."
  cd orangepi-build
  local rev_config="$(git rev-list --count HEAD "${_config}")"
  local id_config="$(git rev-parse --short HEAD:"${_config}")"
  cd ../"${_srcname}"
  local rev_kernel="$(git rev-list --count HEAD)"
  local id_kernel="$(git rev-parse --short HEAD)"
  scripts/setlocalversion --save-scmversion

  echo - > localversion.09-hyphen
  echo "r$(( "${rev_kernel}" + "${rev_config}" ))" > localversion.10-release-total
  echo - > localversion.19-hyphen
  echo "${id_kernel}" > localversion.20-id-kernel
  echo - > localversion.29-hyphen
  echo "${id_config}" > localversion.30-id-config
  echo "-${pkgrel}" > localversion.40-pkgrel
  echo '-aarch64-rk3588-bsp5.10' > localversion.50-pkgname

  # Apply package patches
  local patch=
  for patch in \
    0001-rga3_uncompact_fix.patch \
    0002-vop2_rgba2101010_capability_fix.patch \
    0003-tp_link_ub500.patch
  do
    echo "Applying package provided patch '${patch}'..."
    patch -p1 -N -i ../"${patch}"
  done

  # this is only for local builds so there is no need to integrity check. (if needed)
  for p in "${startdir}"/custom/*.patch; do
    echo "Custom Patching with ${p}"
    patch -p1 -N -i $p || true
  done

  echo "Updating config file..."
  cat "../orangepi-build/${_config}" > '.config'
  make olddefconfig
}

pkgver() {
  cd "${_srcname}"
  printf '%s.%s.%s.%s' \
    "$(make kernelversion)" \
    "$(<localversion.10-release-total)" \
    "$(<localversion.20-id-kernel)" \
    "$(<localversion.30-id-config)"
}

build() {
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Rockchip devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_package() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'wireless-regdb: to set the correct wireless channels of your country'
  )

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove hbuild and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}

  # Install DTB
  echo 'Installing DTBs for Rockchip SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/rockchip"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  # Compiling modules against the tree needs gcc-wrapper.py clang-wrapper.py,
  # which both need Python.
  # Discussed in https://github.com/7Ji/archrepo/issues/5
  # About why depends instead of optdepends, this is a similar decision to
  # https://bugs.archlinux.org/task/69654
  depends=('python')
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done
