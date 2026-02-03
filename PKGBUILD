# Maintainer: Dennis <dmaldana@uvg.edu.gt>
pkgname=aruba-onboard-uvg
pkgver=1.0.0
pkgrel=3
pkgdesc="HPE Aruba Onboard client for UVG CAMPUS CENTRAL (802.1X enrollment)"
arch=('x86_64')
url="https://www.arubanetworks.com/"
license=('custom')
depends=('networkmanager' 'wpa_supplicant' 'qt5-base' 'qt5-svg' 'openssl' 'iw')
conflicts=('iwd')
optdepends=('systemd: wifi-interface service for Framework laptops')
source=("aruba-onboard-data.tar.gz"
        "aruba-onboard-wrapper"
        "aruba-enrollment-helper"
        "wifi-interface.service"
        "aruba-onboard.desktop")
sha256sums=('c41f28d53bbb8fe1a9bbb0de7bd0587d3288090fc00742e1ea15d082122a50eb'
            '0591b3ceb0a8611e2eb46f691ff7157d774182d36fd24f047e55d90a763c4355'
            '6950e06c6746bcc28694ec6addb8d3b74fc00b931d915cf5b0fe7bc162a0d31f'
            '97e696c0ffb540a6b52b95c011be1986ffa7ffb9896fbb4f5eed4ce173e96c77'
            '9ef0cf82134826ec42e060789ece9a1ccce42090a3713cf85ce4ee3626ba5f0a')
install=aruba-onboard-uvg.install

package() {
    # Copy extracted files to package directory
    cd "$srcdir"
    cp -a usr "$pkgdir/"

    # Move binaries to /opt
    install -dm755 "$pkgdir/opt/aruba-onboard/bin"
    mv "$pkgdir/usr/share/aruba-onboard/bin"/* "$pkgdir/opt/aruba-onboard/bin/"

    # Install wrapper scripts
    install -Dm755 "$srcdir/aruba-onboard-wrapper" "$pkgdir/usr/bin/aruba-onboard"
    install -Dm755 "$srcdir/aruba-enrollment-helper" "$pkgdir/usr/lib/aruba-onboard/enrollment-helper"

    # Install desktop file (modified to use wrapper)
    install -Dm644 "$srcdir/aruba-onboard.desktop" "$pkgdir/usr/share/applications/aruba-onboard.desktop"

    # Install icon
    if [ -f "$pkgdir/usr/share/aruba-onboard/main.png" ]; then
        install -Dm644 "$pkgdir/usr/share/aruba-onboard/main.png" "$pkgdir/usr/share/pixmaps/aruba-onboard.png"
    fi

    # Install wifi-interface service (optional, for Framework laptops)
    install -Dm644 "$srcdir/wifi-interface.service" "$pkgdir/usr/lib/systemd/system/wifi-interface.service"

    # Keep /usr/share/aruba-onboard directory (needed for runtime lock files)
    # Remove only the bin/ subdirectory we already moved
    rm -rf "$pkgdir/usr/share/aruba-onboard/bin"

    # Ensure the directory exists with writable permissions for lock files
    # Mode 1777 = sticky bit + world writable (like /tmp)
    install -dm1777 "$pkgdir/usr/share/aruba-onboard"

    # Clean up doc files
    rm -rf "$pkgdir/usr/share/doc"
}
