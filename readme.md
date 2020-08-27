we're running macos 10.14.6, with gcc 4.2.1. before setting up our cross
toolchain, let's grab latest gcc (10.2.0):

```sh
mkdir gccbuild
cd gccbuild
../gcc-10.2.0/configure \
--prefix=/usr/local/gcc-10.2.0 \
--program-suffix=-10.2 \
--enable-checking=release \
--with-sysroot=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk

# this took 16? hours on my poor lil laptop :'-)
make -j8
sudo make install-strip
```

and use it:

```sh
export CC=/usr/local/gcc-10.2.0/bin/gcc-10.2
export CXX=/usr/local/gcc-10.2.0/bin/g++-10.2
export CPP=/usr/local/gcc-10.2.0/bin/cpp-10.2
export LD=/usr/local/gcc-10.2.0/bin/gcc-10.2
```

next, set a home for our cross builds, so that they're isolated from system
tools, and what we're targeting:

```sh
export PREFIX=/usr/local/cross
export PATH="$PREFIX/bin:$PATH"
export TARGET=i686-elf
```

now we build binutils for our cross compiler. do this before building gcc,
which presumably links target libraries--we ran into issues when we tried using
gcc built without cross binutils:

```sh
mkdir crossbin
cd crossbin
../binutils-2.35/configure \
--target=$TARGET \
--prefix="$PREFIX" \
--with-sysroot \
--disable-nls \
--disable-werror

make -j8
sudo make install
```

and the cross compiler:

```sh
cd gcc-10.2.0
./contrib/download_prequisites
cd ..
mkdir crossgcc
cd crossgcc
../gcc-10.2.0/configure \
--prefix="$PREFIX" \
--target=$TARGET \
--disable-nls \
--enable-languages=c,c++ \
--without-headers \
--with-sysroot=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk

make -j8 all-gcc
make -j8 all-target-libgcc
sudo make install-gcc
sudo make install-target-libgcc
```

we'll be building our boot images with grub:

```sh
# grub needs xorriso and objconv
brew install xorriso

mkdir objconvbuild
/usr/local/gcc-10.2.0/bin/g++-10.2 -o objconv -O2 src/*.cpp --prefix="$PREFIX"
cp objconv "$PREFIX/bin"
cd ..

cd grub-2.04
./autogen.sh
# we had to make this 2-line patch to get 2.04 to build:
# https://www.mail-archive.com/grub-devel@gnu.org/msg29007.html
cd ..
mkdir grubbuild
cd grubbuild
../grub-2.04/configure --prefix="$PREFIX" --target=$TARGET --disable-werror
```
