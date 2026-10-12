# Building WYSIWYG-Editor on Windows

This guide describes how to build the WYSIWYG-Editor on Windows by using
Ubuntu in WSL (Windows Subsystem for Linux). The build produces a Docker image
containing the compiled editor and its LibreOffice-based engine.

## Prerequisites

Before starting, make sure the following tools are available:

- Windows with WSL and Docker Desktop installed.
- An Ubuntu distribution supported by the repository's Docker base image.
- Internet access for downloading packages and source repositories.
- Enough disk space for the source tree, build artifacts, and Docker image.

The Ubuntu version used for WSL may change over time. Check
`docker/from-source/Ubuntu` and use the distribution version expected by that
image. The commands below use Ubuntu 24.04 as an example.

## Install Ubuntu in WSL

Install Ubuntu from an elevated PowerShell or Command Prompt. After the first
launch, exit and restart Ubuntu so that WSL opens in the expected working
directory.

```powershell
wsl --install Ubuntu-24.04
```

## Update Ubuntu

Update the package index and upgrade packages before installing the build
dependencies.

```bash
sudo apt update -y
sudo apt upgrade -y
```

## Install build dependencies

The `dialog` package is required by some package installation steps that may
otherwise fail when no interactive dialog frontend is available.

```bash
sudo apt install -y dialog
```

Install the dependencies required to build the LibreOffice-based engine and
the WYSIWYG-Editor.

```bash
sudo apt-get install -y \
  git build-essential zip ccache junit4 libkrb5-dev nasm graphviz \
  python3 python3-dev python3-setuptools qtbase5-dev \
  libkf5coreaddons-dev libkf5i18n-dev libkf5config-dev \
  libkf5windowsystem-dev libkf5kio-dev libqt5x11extras5-dev \
  autoconf libcups2-dev libfontconfig1-dev gperf openjdk-17-jdk doxygen \
  libxslt1-dev xsltproc libxml2-utils libxrandr-dev libx11-dev bison flex \
  libgtk-3-dev libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev \
  ant ant-optional libnss3-dev libavahi-client-dev libxt-dev meson \
  libarchive-zip-perl
```

## Enable Ubuntu source packages

The LibreOffice build dependencies are installed from Ubuntu source package
metadata. Enable source repositories in the Ubuntu sources configuration.

Open the configuration file:

```bash
sudo nano /etc/apt/sources.list.d/ubuntu.sources
```

Find the Ubuntu repository entry. It should look similar to this:

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
```

Change only the `Types` line to include source packages:

```text
Types: deb deb-src
```

In `nano`, save with `Ctrl+O`, press `Enter`, and exit with `Ctrl+X`.

Refresh the package index so APT recognizes the newly enabled source
repositories:

```bash
sudo apt update -y
```

Install LibreOffice's declared build dependencies:

```bash
sudo apt build-dep -y libreoffice
```

Install the additional dependencies used by the WYSIWYG-Editor build and
image creation steps:

```bash
sudo apt install -y \
  python3-polib libcap-dev libssl-dev npm libpam-dev libzstd-dev wget \
  git build-essential libtool libcap2-bin python3-lxml libpng-dev \
  libgif-dev libcppunit-dev pkg-config fontconfig snapd chromium-browser
```

## Troubleshoot an outdated Node.js version

If a later build step fails because the installed Node.js version is too old,
install Node Version Manager (nvm) and switch to the latest available LTS
release. Skip the installation command if `nvm` is already installed.

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc

nvm install --lts
nvm alias default 'lts/*'
nvm use --lts
```

Verify that the shell is using the updated versions before rerunning the
failed build step:

```bash
node --version
npm --version
```

## Clone the source repositories

Clone the WYSIWYG-Editor repository and download the translation repository
required by the engine build.

```bash
git clone https://github.com/apenio-GmbH/WYSIWYG-Editor.git
cd WYSIWYG-Editor/engine
git clone https://gerrit.collaboraoffice.com/translations
```

## Build the engine

Configure and compile the LibreOffice-based engine. The configuration targets
the `CPLinux-LOKit` distribution, builds German resources, skips package
creation, and disables debug symbols to reduce the build output.

```bash
./autogen.sh \
  --with-distro=CPLinux-LOKit \
  --with-lang=de \
  --disable-epm \
  --without-package-format \
  --disable-symbols

make -j "$(nproc)"
```

## Build the WYSIWYG-Editor

Return to the repository root, configure the editor to use the engine built in
the previous step, and compile it.

```bash
cd ..

./autogen.sh

./configure \
  --prefix=/usr \
  --sysconfdir=/etc \
  --localstatedir=/var \
  --disable-tests \
  --with-lokit-path="$(pwd)/engine/include" \
  --with-lo-path=/opt/collaboraoffice \
  --disable-about

make -j "$(nproc)"
```

## Prepare the Docker build context

Docker Desktop must be integrated with the Ubuntu distribution used for the
build. In Docker Desktop, open **Settings > Resources > WSL Integration** and
enable integration for `Ubuntu-24.04` (or the distribution selected above).

Copy the compiled engine and the installed editor into the Docker build
context. The first command removes any artifacts from an earlier build.

```bash
rm -rf docker/from-source/instdir
mkdir -p docker/from-source/instdir/opt/

cp -a engine/instdir docker/from-source/instdir/opt/collaboraoffice
DESTDIR="$(realpath docker/from-source/instdir)" make install
```

## Build the Docker image

Build the image from the repository's Ubuntu Dockerfile and tag it as
`apenio/wysiwyg-editor:1.0`.

```bash
cd docker/from-source
docker build --no-cache -t apenio/wysiwyg-editor:1.0 -f Ubuntu .
```

The resulting image is available locally under the tag
`apenio/wysiwyg-editor:1.0`.