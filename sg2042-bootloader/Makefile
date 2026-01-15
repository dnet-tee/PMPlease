#TARGETS = debian-kernel linuxboot-kernel opensbi pack sdcard-image u-root zsbl
#
#all: $(TARGETS)
#$(TARGETS):
#	docker build --output=out --target $@ .
#
#.PHONY: $(TARGETS)
TARGETS = debian-kernel linuxboot-kernel opensbi pack sdcard-image u-root zsbl

ROOT_DIR := $(abspath ..)

all: $(TARGETS)

$(TARGETS):
	docker build \
		-f $(ROOT_DIR)/sg2042-bootloader/Dockerfile \
		--output=$(ROOT_DIR)/sg2042-bootloader/out \
		--target $@ $(ROOT_DIR)

.PHONY: $(TARGETS)

