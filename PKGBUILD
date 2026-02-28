# Maintainer: stonespren <your-email@example.com>
pkgname=helium-profile-sync
pkgver=1.0.0
pkgrel=1
pkgdesc="Automatically sync Helium browser profiles to AWS S3 using rclone"
arch=('any')
url="https://github.com/stonespren/helium-profile-sync"
license=('MIT')
depends=(
    'rclone'
    'aws-cli-v2'
    'jq'
    'bash'
    'systemd'
)
optdepends=(
    'helium-browser: The browser whose profiles are synced'
)
source=(
    'helium-profile-sync'
    'helium-profile-sync-setup'
    'helium-profile-sync.service'
    'helium-profile-sync.timer'
    'helium-profile-sync.conf.example'
    'LICENSE'
)
# Run 'updpkgsums' or 'makepkg -g' to generate real checksums before publishing
sha256sums=('SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP' 'SKIP')

package() {
    # ── Scripts ───────────────────────────────────────────────
    install -Dm755 "$srcdir/helium-profile-sync" \
        "$pkgdir/usr/bin/helium-profile-sync"

    install -Dm755 "$srcdir/helium-profile-sync-setup" \
        "$pkgdir/usr/bin/helium-profile-sync-setup"

    # ── systemd user units ────────────────────────────────────
    install -Dm644 "$srcdir/helium-profile-sync.service" \
        "$pkgdir/usr/lib/systemd/user/helium-profile-sync.service"

    install -Dm644 "$srcdir/helium-profile-sync.timer" \
        "$pkgdir/usr/lib/systemd/user/helium-profile-sync.timer"

    # ── Documentation / example config ────────────────────────
    install -Dm644 "$srcdir/helium-profile-sync.conf.example" \
        "$pkgdir/usr/share/doc/$pkgname/helium-profile-sync.conf.example"

    install -Dm644 "$srcdir/LICENSE" \
        "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}