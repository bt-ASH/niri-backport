# Maintainer: David Runge <dvzrv@archlinux.org>
# Maintainer: Caleb Maclennan <caleb@alerque.com>
# Maintainer: Robin Candau <antiz@archlinux.org>

pkgname=niri
pkgver=26.04
pkgrel=1.1
pkgdesc="A scrollable-tiling Wayland compositor"
arch=(x86_64)
url="https://github.com/niri-wm/niri"
license=(GPL-3.0-or-later)
depends=(
  cairo
  glib2
  glibc
  libdisplay-info
  libgcc
  libinput
  libpipewire
  libxkbcommon
  mesa
  pango
  pixman
  seatd
  systemd-libs
  xdg-desktop-portal-impl
)
makedepends=(
  clang
  git
  rust
)
optdepends=(
  'alacritty: a suggested GPU-accelerated terminal emulator'
  'bash: for niri-session script'
  'fuzzel: a suggested Wayland application launcher'
  'mako: a suggested Wayland notification daemon'
  'org.freedesktop.secrets: for apps to rely on secrets portal'
  'swaybg: a suggested Wayland wallpaper tool'
  'swaylock: a suggested Wayland screen locker'
  'waybar: a suggested Wayland customizable desktop bar'
  'xwayland-satellite: for running X11 apps in XWayland'
  'xdg-desktop-portal-gtk: a suggested XDG desktop portal'
  'xdg-desktop-portal-gnome: a XDG desktop portal required for screencasting'
)
provides=(wayland-compositor)
source=($pkgname-$pkgver.tar.gz::$url/archive/refs/tags/v$pkgver.tar.gz
        $pkgname-shm-sharing.patch)
sha512sums=('ae30446fbf529155907726cf2cba7402bcfc5be2032370d66f9f980efb257f29353b47d495f1218b81ac32c643f15bbb74834dc35882849a4479021b4b9fa97f'
            'SKIP')
b2sums=('65fc4df85ab40c1a7a82900a9812c542a8a00467a0f1742f2dede50c0e323bd8957149e91b181a6c71ad077735f6a4db53451bbad0deb768f7d4840ebe798a20'
        'SKIP')

prepare() {
  cd $pkgname-$pkgver
  # PR #1791: support SHM sharing in screencast (fixes "no more input formats")
  patch -Np1 -i "$srcdir/$pkgname-shm-sharing.patch"
  cargo fetch --locked --target "$(rustc --print host-tuple)"
}

build() {
  cd $pkgname-$pkgver
  export NIRI_BUILD_COMMIT="$(zcat ../$pkgname-$pkgver.tar.gz | git get-tar-commit-id | cut -c1-7)"
  CFLAGS+=(' -ffat-lto-objects')
  cargo build --frozen --release --features default

  # generate shell completions
  for shell in bash fish zsh; do
    cargo run --frozen --release --bin niri -- \
      completions "$shell" > "$shell-completions"
  done
}

check() {
  cd "$pkgname-$pkgver"
  export XDG_RUNTIME_DIR="$(mktemp -d)"
  export RAYON_NUM_THREADS=1  # required so we can build in environments with _many_ threads
  cargo test --all --exclude niri-visual-tests --frozen
}

package() {
  cd $pkgname-$pkgver
  install -vDm 755 {target/release/$pkgname,resources/$pkgname-session} -t "$pkgdir/usr/bin/"
  install -vDm 644 resources/$pkgname{.service,-shutdown.target} -t "$pkgdir/usr/lib/systemd/user/"
  install -vDm 644 resources/$pkgname.desktop -t "$pkgdir/usr/share/wayland-sessions/"
  install -vDm 644 resources/$pkgname-portals.conf -t "$pkgdir/usr/share/xdg-desktop-portal/"
  install -vDm 644 resources/default-config.kdl README.md -t "$pkgdir/usr/share/doc/$pkgname/"
  # shell auto-completions
  install -vDm 644 bash-completions "$pkgdir/usr/share/bash-completion/completions/niri"
  install -vDm 644 fish-completions "$pkgdir/usr/share/fish/vendor_completions.d/niri.fish"
  install -vDm 644 zsh-completions "$pkgdir/usr/share/zsh/site-functions/_niri"

}
