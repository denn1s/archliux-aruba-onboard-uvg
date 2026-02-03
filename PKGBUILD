# Maintainer: Dennis <dmaldana@uvg.edu.gt>
pkgname=aruba-onboard-uvg
pkgver=1.0.0
pkgrel=1
pkgdesc="HPE Aruba Onboard client for UVG CAMPUS CENTRAL (802.1X enrollment)"
arch=('x86_64')
url="https://www.arubanetworks.com/"
license=('custom')
depends=('networkmanager' 'wpa_supplicant' 'qt5-base' 'qt5-svg' 'openssl' 'iw')
conflicts=('iwd')
optdepends=('systemd: wifi-interface service for Framework laptops')
source=("Aruba_Onboard_Installer.deb"
        "aruba-onboard-wrapper"
        "aruba-enrollment-helper"
        "wifi-interface.service"
        "aruba-onboard.desktop")
sha256sums=('SKIP'  # Will be filled after creating all files
            'SKIP'
            'SKIP'
            'SKIP'
            'SKIP')
install=aruba-onboard-uvg.install

package() {
    # Extract the .deb
    cd "$srcdir"
    ar x Aruba_Onboard_Installer.deb
    tar xf data.tar.xz -C "$pkgdir"

    # Fix /lib -> /usr/lib (Arch standard)
    if [ -d "$pkgdir/lib" ]; then
        install -dm755 "$pkgdir/usr/lib"
        cp -a "$pkgdir/lib"/* "$pkgdir/usr/lib/"
        rm -rf "$pkgdir/lib"
    fi

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

    # Ensure the directory exists even if empty
    install -dm755 "$pkgdir/usr/share/aruba-onboard"

    # Clean up doc files
    rm -rf "$pkgdir/usr/share/doc"
}
