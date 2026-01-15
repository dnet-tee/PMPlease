################################################################################
# SOURCE
################################################################################

FROM scratch AS linux-src
ARG LINUX_URL=https://github.com/sophgo/linux-riscv.git
# 6.6.20: caa949e3690fe8a4656313b2b56f52666fa880db
# 6.6.66: c30fe9b9c41f6423d9032698208c2dc3865b6536
ARG LINUX_COMMIT=c30fe9b9c41f6423d9032698208c2dc3865b6536
ADD --keep-git-dir=true "${LINUX_URL}#${LINUX_COMMIT}" /linux
ADD sg2042-bootloader/linux-bootargs.patch /linux
ADD sg2042-bootloader/linux-cmdline.patch /linux

FROM scratch AS opensbi-src
ARG OPENSBI_URL=https://github.com/sophgo/opensbi.git
ARG OPENSBI_COMMIT=506f47f9fa18224a0e6a642aff712d2547d61139
ADD "${OPENSBI_URL}#${OPENSBI_COMMIT}" /opensbi

FROM scratch AS u-root-src
ARG UROOT_URL=https://github.com/u-root/u-root.git
ARG UROOT_COMMIT=v0.14.0
ADD "${UROOT_URL}#${UROOT_COMMIT}" /u-root

FROM scratch AS zsbl-src
#copy local fork
COPY milkv-zsbl/ /zsbl 

#ARG ZSBL_URL=https://github.com/sophgo/zsbl.git
##ARG ZSBL_COMMIT=cc806273e0f679bef2f6b017c68adede1594ad31
#ARG ZSBL_COMMIT=018c9a021f455c24912dd5980448257271f50972
#ADD "${ZSBL_URL}#${ZSBL_COMMIT}" /zsbl

################################################################################
# BASE
################################################################################

FROM ubuntu:24.04 AS base

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
&& apt install -yq --no-install-recommends \
      dosfstools build-essential \
      libncurses-dev gawk flex bison openssl libssl-dev tree \
      dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf \
      device-tree-compiler xz-utils \
      curl automake autotools-dev python3 libmpc-dev libmpfr-dev \
      libgmp-dev texinfo gperf libtool patchutils bc zlib1g-dev \
      libexpat-dev ninja-build uuid-dev \
      git ca-certificates bsdextrautils \
      cpio debhelper devscripts rsync \
      golang mtools parted

# Cross compile toolchain
ARG MAINLINE_TOOLCHAIN=https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2025.05.16
ARG MAINLINE_TOOLCHAIN_FILENAME=riscv64-glibc-ubuntu-24.04-gcc-nightly-2025.05.16-nightly.tar.xz

RUN curl -L "${MAINLINE_TOOLCHAIN}/${MAINLINE_TOOLCHAIN_FILENAME}" | tar -Jxv -C /opt
ENV PATH="/opt/riscv/bin:$PATH"
ENV CROSS_COMPILE=riscv64-unknown-linux-gnu-

# Linux stuff
ENV ARCH=riscv
ENV KBUILD_BUILD_USER=riscv
ENV KBUILD_BUILD_HOST=riscv-builder
ENV KDEB_COMPRESS=xz

################################################################################
# DEBIAN KERNEL
################################################################################

FROM base AS debian-kernel-build
COPY --from=linux-src /linux /linux
WORKDIR /linux
RUN patch -Np1 -i linux-bootargs.patch
RUN patch -Np1 -i linux-cmdline.patch

# Copy defconfig on directory
COPY sg2042-bootloader/revyos_cma_defconfig /linux/arch/riscv/configs/ 
# make kernel with custom cma config
RUN make revyos_cma_defconfig
#RUN make revyos_defconfig
RUN make -j$(nproc) dtbs
RUN make -j$(nproc) bindeb-pkg \
  KDEB_PKGVERSION="$(git log -1 --date=format:'%Y%m%d' --format='%cd+%h')" \
  LOCALVERSION="-pioneer-cma"

FROM scratch AS debian-kernel
COPY --from=debian-kernel-build /linux/arch/riscv/boot/dts/sophgo/*.dtb /
COPY --from=debian-kernel-build /*.deb /

################################################################################
# LINUXBOOT KERNEL
################################################################################

FROM base AS linuxboot-kernel-build
COPY --from=linux-src /linux /linux
WORKDIR /linux
RUN patch -Np1 -i linux-bootargs.patch
RUN make -j$(nproc) sophgo_mango_normal_defconfig
RUN make -j$(nproc)

FROM scratch AS linuxboot-kernel
COPY --from=linuxboot-kernel-build /linux/arch/riscv/boot/Image /riscv64_Image
COPY --from=linuxboot-kernel-build /linux/arch/riscv/boot/dts/sophgo/mango-*.dtb /

################################################################################
# OPENSBI
################################################################################

FROM base AS opensbi-build
COPY --from=opensbi-src /opensbi /opensbi
WORKDIR /opensbi
RUN make -j$(nproc) PLATFORM="generic" FW_PIC=y BUILD_INFO=y DEBUG=1 FW_DEBUG=1 FW_OPTIONS=2

FROM scratch AS opensbi
COPY --from=opensbi-build /opensbi/build/platform/generic/firmware/fw_dynamic.bin /

################################################################################
# U-ROOT
################################################################################

# FROM golang:1.22 AS u-root-build
FROM base AS u-root-build
COPY --from=u-root-src /u-root /u-root
WORKDIR /u-root
RUN go build
ENV GOOS=linux GOARCH=riscv64
RUN /u-root/u-root -build bb -uinitcmd=boot -o /initrd.img core boot 

FROM scratch AS u-root
COPY --from=u-root-build /initrd.img /

################################################################################
# ZSBL
################################################################################

FROM base AS zsbl-build
COPY --from=zsbl-src /zsbl /zsbl
WORKDIR /zsbl
RUN make sg2042_defconfig
RUN make -j$(nproc)

FROM scratch AS zsbl
COPY --from=zsbl-build /zsbl/zsbl.bin /

################################################################################
# FIRMWARE PACK
################################################################################

FROM base AS pack-build
ADD https://github.com/sophgo/bootloader-riscv.git#:scripts/pack /pack
WORKDIR /pack
RUN make

FROM scratch AS pack
COPY --from=pack-build /pack/pack /

################################################################################
# SDCARD IMAGE
################################################################################

FROM base AS sdcard-image-build

ADD https://github.com/sophgo/bootloader-riscv/raw/c8602a4d91c04f3855d2397f946bd0b64aaa7119/firmware/fip.bin /firmware/fip.bin
COPY --from=zsbl /zsbl.bin /firmware/zsbl.bin
#COPY --from=opensbi /fw_dynamic.bin /firmware/riscv64/fw_dynamic.bin
COPY sg2042-bootloader/fw_dynamic.bin /firmware/riscv64/fw_dynamic.bin
COPY --from=u-root /initrd.img /firmware/riscv64/initrd.img
COPY --from=linuxboot-kernel /mango-milkv-pioneer.dtb /firmware/riscv64/mango-milkv-pioneer.dtb
COPY --from=linuxboot-kernel /riscv64_Image /firmware/riscv64/riscv64_Image

# Based on https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/installer/sd-card/sd-image.nix
ARG FIRMWARE_PARTITION_NAME=SG2042
ARG FIRMWARE_PARTITION_OFFSET=1
ARG FIRMWARE_SIZE=48

RUN <<EOR
set -eux

export img=sdcard.img

# Gap in front of the first partition, in MiB
gap=${FIRMWARE_PARTITION_OFFSET}

# Create the image file sized to fit /boot/firmware, plus slack for the gap.
firmwareSizeBlocks=$((${FIRMWARE_SIZE} * 1024 * 1024 / 512))
imageSize=$((firmwareSizeBlocks * 512 + gap * 1024 * 1024))
truncate -s $imageSize $img

parted $img --script mklabel msdos
parted $img --script mkpart primary fat32 ${FIRMWARE_PARTITION_OFFSET}MiB 100%

# Create a FAT32 /boot/firmware partition of suitable size into firmware_part.img
eval $(partx $img -o START,SECTORS --nr 1 --pairs)
truncate -s $((SECTORS * 512)) firmware_part.img

mkfs.vfat --invariant -n ${FIRMWARE_PARTITION_NAME} firmware_part.img
  
# Copy the populated /boot/firmware into the SD image
cd firmware
# Force a fixed order in mcopy for better determinism, and avoid file globbing
for d in $(find . -type d -mindepth 1 | sort); do
  mmd -i ../firmware_part.img "::/$d"
done
for f in $(find . -type f | sort); do
  mcopy -pvm -i ../firmware_part.img "$f" "::/$f"
done
cd ..

# Verify the FAT partition before copying it.
fsck.vfat -vn firmware_part.img
dd conv=notrunc if=firmware_part.img of=$img seek=$START count=$SECTORS
EOR

FROM scratch AS sdcard-image
COPY --from=sdcard-image-build /sdcard.img /
