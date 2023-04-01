# Signal Desktop Builder
This project allows building Signal Desktop for Debian 11 on ARM64.
It is currently a work in progress, with the goal of building a flatpak
which provides Signal Desktop.

This repository is a fork of [undef1/signal-desktop-builder](https://gitlab.com/undef1/signal-desktop-builder)

## Installing my Flatpak build

For directions on installing the flatpak, seek [here](https://elagost.com/flatpak).

## Installing via Deb

The upstream repo provides .deb binaries [here](https://gitlab.com/undef1/signal-desktop-builder/-/packages) for some releases.

## Building Signal

The process works through CI fairly well. I've included all the files in this repository for each of the CI platforms I've made it work on.

- `.build.yml` will work for [sourcehut builds](https://builds.sr.ht).
- `.gitlab-ci.yml` is obviously for [gitlab](https://gitlab.com) ci but is kind of incomplete, since I ran it on a self-hosted runner.
- `.github/workflows/build.yml` is for [github](https://github.com) actions.

To build by hand, you will need an Ubuntu or Debian server.

### Installing dependencies

This needs to be done every time on CI, but only once on a self-hosted system. You can use docker instead of podman but will need to modify the scripts or set aliases yourself.

```
sudo apt install -qq bash rsync podman flatpak elfutils coreutils slirp4netns rootlesskit binfmt-support fuse-overlayfs flatpak-builder qemu-user-static
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
sudo flatpak install --noninteractive --arch=aarch64 flathub org.electronjs.Electron2.BaseApp//22.08 org.freedesktop.Platform//22.08 org.freedesktop.Sdk//22.08 -y
```

### Running Build Scripts

Fairly simple. `ci-build.sh` invokes `signal-buildscript.sh`, builds signal in an ARM docker container and copies the .deb out. It looks like there's some duplication of work between them; for historical reasons this was necessary because one of them would fail due to non-interactivity. I think if you run it by hand in tmux or something, you can comment out most of `ci-build.sh`.

First, though, run `./update-node.sh 6.12.x` where `6.12.x` is the name of the branch you are building. If you get a new nodejs version, update the Dockerfile's `ENV NODE_VERSION` line.

```
export VERSION="6.12.0"
bash ci-build.sh
podman stop signal-desktop-$VERSION
cp ~/signal-desktop.deb .
```
This build takes 2-3 hours.

If you just want the .deb, you now have it. Congrats.

## Building a Flatpak

Flatpak repos are just flat directories and a .flatpakrepo file.

You'll need a GPG key - if it's password protected you'll get asked for the password when building so you can't do that to a key you use in CI. To make one use `gpg --gen-key`. You don't have to give it your "real" info.

The flatpakrepo file looks like this:

```
[Flatpak Repo]
Title=Signal-Arm Flatpak Repo
Url=https://example.com/flatpak/signal-arm-repo/
GPGKey=<Key Data>
```

To get the key data, run `gpg --armor --export <key email or ID> > key.gpg`. 

Before you send that key anywhere, inspect `key.gpg` and make sure it begins and ends with `PGP PUBLIC KEY BLOCK` and __NOT__ `PGP PRIVATE KEY BLOCK`. Your private key should be kept private.

If you've made sure it's a public key, run `base64 --wrap=0 < key.gpg`. This is the key you put in `<Key Data>`.

For more info see [Flatpak.org's documentation on hosting a repo](https://docs.flatpak.org/en/latest/hosting-a-repository.html).

Get the Key ID of your secret key. You can get it in the GNOME application "Passwords and Keys" (or `seahorse`), or `gpg --list-keys --keyid-format long`.

Look for this line and that's the ID you supply to flatpak-builder.

```
pub   rsa4096/FBEF43DC8C6BE9A7 2022-06-04 [SC]
             |-- ^ this ID ---|
```

Build the flatpak:

```
flatpak-builder --arch=aarch64 --gpg-sign=<Key ID> --repo=./repodir --force-clean ./builddir flatpak.yml
```

Now you have your `.flatpakrepo` file and your `./repodir`. You can put those on a web server and tell people about them, or use them yourself.

## Github Actions notes:

to publish a new release:

- find latest stable tag from [upstream repo](https://github.com/signalapp/Signal-Desktop/releases)
- edit `Dockerfile` and set git clone to pull the correct branch (format is "5.45.x")
- run `update-node 5.45.x` to set Dockefile to use upstream's specified nodejs version
    - for '5.45.1' or other versions, you use the same '5.45.x' argument.
- update `org.signal.Signal.metainfo.xml` with the new version
- edit `.builds.yml` and set `environment.VERSION` to new version
- push changes and sourcehut builds should trigger a new build

## See also:
https://github.com/lsfxz/ringrtc/tree/aarch64  
https://gitlab.com/undef1/Snippets/-/snippets/2100495  
https://gitlab.com/ohfp/pinebookpro-things/-/tree/master/signal-desktop  
Flatpak based on [Flathub Sigal Desktop builds](https://github.com/flathub/org.signal.Signal/)
 - `signal-desktop.sh` https://github.com/flathub/org.signal.Signal/blob/master/signal-desktop.sh
 - `org.signal.Signal.metainfo.xml` https://github.com/flathub/org.signal.Signal/blob/master/org.signal.Signal.metainfo.xml
 - `flatpak.yml` https://github.com/flathub/org.signal.Signal/blob/master/org.signal.Signal.yaml
