# Maintainer: David Runge <dvzrv@archlinux.org>
# Contributor:  Joakim Hernberg <jbh@alchemy.lu>

# Package informations
pkgbase=linux-rt-lts
pkgver=5.10.27.36.arch1
pkgrel=1
pkgdesc='Linux RT LTS'
arch=('x86_64')
url="https://wiki.linuxfoundation.org/realtime/start"
license=('GPL2')
makedepends=('wget' 'bc' 'git' 'graphviz' 'imagemagick' 'kmod' 'libelf' 'pahole' 'python-sphinx' 'python-sphinx_rtd_theme' 'xmlto' 'clang' 'llvm' 'modprobed-db')
options=('!strip')

source=(
  "https://gitlab.archlinux.org/dvzrv/linux-rt-lts/-/archive/v5.10.27.36.arch1/linux-rt-lts-v5.10.27.36.arch1.tar.gz"
  'config'
)

validpgpkeys=(
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman <gregkh@linuxfoundation.org>
  '5ED9A48FC54C0A22D1D0804CEBC26CDB5A56DE73'  # Steven Rostedt (Der Hacker) <rostedt@goodmis.org>
  'C7E7849466FE2358343588377258734B41C31549'  # David Runge <dvzrv@archlinux.org>
)

# Build vars
export KBUILD_BUILD_HOST=$(cat /etc/hostname)
export KBUILD_BUILD_USER=$USER
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
makeargs="-j$(nproc --all) CC=clang HOSTCC=clang"

# Custom vars
_htmldocs=n
_localmodcfg=y

# Prepare Function
prepare()
{
  # Renaming dir since new method
  mv ""$pkgbase"-v"$pkgver"" "$pkgbase" 

  # Enter kernel dir
  cd "${pkgbase}"

  # Setting up kernel version
  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  # Applying RT patches
  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    src="${src//patch.xz/patch}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  # Generating config
  echo "Setting config..."
  cp ../config .config
  make olddefconfig

  # Using only needed modules
  if [[ "$_localmodcfg" == 'y' ]]; then
    if [[ $HOME/.config/modprobed.db ]]; then
      rm $HOME/.config/modprobed-db.conf $HOME/.config/modprobed.db
    fi
    modprobed-db && modprobed-db storesilent
    echo "Running Steven Rostedt's make localmodconfig now"
    make LSMOD=$HOME/.config/modprobed.db localmodconfig
  fi

  # Done
  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

# Build function
build()
{
  # Enter kernel dir
  cd "${pkgbase}"

  # Building kernel
  make all $makeargs

  # Building htmldocs
  if [[ "$_htmldocs" == 'y' ]]; then
    make htmldocs
  fi
}

_package()
{
  # Show package informations
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils initramfs kmod)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)

  # Entering kernel dir
  cd "${pkgbase}"

  # Set up local environments
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  # Installing boot image
  # Note:
  #       systemd expects to find the kernel here to allow hibernation
  #       https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  echo "Installing boot image..."
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  # Installing modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # Remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers()
{
  # Showing package informations
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"

  # Entering kernel dir
  cd "${pkgbase}"

  # Set up local environment
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  # Installing build files
  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # Add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # Add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  # Installing headers
  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s
  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h
  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h
  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # Installing KConfig
  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  # Remove uneeded architectures
  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  # Remove documentation
  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  # Remove broken symlinks
  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  # Remove loose objects
  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  # Stripping build tools
  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)           # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)             # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)          # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*)      # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  # Stripping vmlinux
  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  # Create symlink
  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs()
{
  # Showing package informations
  pkgdesc="Documentation for the $pkgdesc kernel"

  # Entering kernel dir
  cd "${pkgbase}"

  # Set up local environtment
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  # Installing documentation
  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  # Create symlink
  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers" "$pkgbase-docs")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
