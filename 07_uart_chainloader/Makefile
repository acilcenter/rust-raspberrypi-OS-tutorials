## SPDX-License-Identifier: MIT
##
## Copyright (c) 2018-2019 Andre Richter <andre.o.richter@gmail.com>

# Default to the RPi3
ifndef BSP
	BSP = rpi3
endif

# BSP-specific arguments
ifeq ($(BSP),rpi3)
	TARGET = aarch64-unknown-none-softfloat
	OUTPUT = kernel8.img
	QEMU_BINARY = qemu-system-aarch64
	QEMU_MACHINE_TYPE = raspi3
	QEMU_MISC_ARGS = -serial stdio
	LINKER_FILE = src/bsp/rpi/link.ld
	RUSTC_MISC_ARGS = -C target-cpu=cortex-a53 -C relocation-model=pic
	CHAINBOOT_DEMO_PAYLOAD = demo_payload_rpi3.img
else ifeq ($(BSP),rpi4)
	TARGET = aarch64-unknown-none-softfloat
	OUTPUT = kernel8.img
#	QEMU_BINARY = qemu-system-aarch64
#	QEMU_MACHINE_TYPE =
#	QEMU_MISC_ARGS = -serial stdio
	LINKER_FILE = src/bsp/rpi/link.ld
	RUSTC_MISC_ARGS = -C target-cpu=cortex-a72 -C relocation-model=pic
	CHAINBOOT_DEMO_PAYLOAD = demo_payload_rpi4.img
endif

SOURCES = $(wildcard **/*.rs) $(wildcard **/*.S) $(wildcard **/*.ld)

XRUSTC_CMD = cargo xrustc \
		--target=$(TARGET) \
		--features bsp_$(BSP) \
		--release \
		-- \
		-C link-arg=-T$(LINKER_FILE) \
		$(RUSTC_MISC_ARGS)

CARGO_OUTPUT = target/$(TARGET)/release/kernel

OBJCOPY_CMD = cargo objcopy \
		-- \
		--strip-all \
		-O binary

CONTAINER_UTILS   = rustembedded/osdev-utils

DOCKER_CMD        = docker run -it --rm
DOCKER_ARG_CURDIR = -v $(shell pwd):/work -w /work
DOCKER_ARG_TTY    = --privileged -v /dev:/dev

DOCKER_EXEC_QEMU         = $(QEMU_BINARY) -M $(QEMU_MACHINE_TYPE) -kernel $(OUTPUT)
DOCKER_EXEC_RASPBOOT     = raspbootcom
DOCKER_EXEC_RASPBOOT_DEV = /dev/ttyUSB0
# DOCKER_EXEC_RASPBOOT_DEV = /dev/ttyACM0

.PHONY: all doc qemu qemuasm chainboot clippy clean readelf objdump nm

all: clean $(OUTPUT)

$(CARGO_OUTPUT): $(SOURCES)
	RUSTFLAGS="-D warnings -D missing_docs" $(XRUSTC_CMD)

$(OUTPUT): $(CARGO_OUTPUT)
	cp $< .
	$(OBJCOPY_CMD) $< $(OUTPUT)

doc:
	cargo xdoc --target=$(TARGET) --features bsp_$(BSP) --document-private-items
	xdg-open target/$(TARGET)/doc/kernel/index.html

ifeq ($(QEMU_MACHINE_TYPE),)
qemu:
	@echo "This board is not yet supported for QEMU."

qemuasm:
	@echo "This board is not yet supported for QEMU."
else
qemu: all
	$(DOCKER_CMD) $(DOCKER_ARG_CURDIR) $(CONTAINER_UTILS) \
	$(DOCKER_EXEC_QEMU) $(QEMU_MISC_ARGS)

qemuasm: all
	$(DOCKER_CMD) $(DOCKER_ARG_CURDIR) $(CONTAINER_UTILS) \
	$(DOCKER_EXEC_QEMU) -d in_asm
endif

chainboot:
	$(DOCKER_CMD) $(DOCKER_ARG_CURDIR) $(DOCKER_ARG_TTY) \
	$(CONTAINER_UTILS) $(DOCKER_EXEC_RASPBOOT) $(DOCKER_EXEC_RASPBOOT_DEV) \
	$(CHAINBOOT_DEMO_PAYLOAD)

clippy:
	cargo xclippy --target=$(TARGET) --features bsp_$(BSP)

clean:
	cargo clean

readelf:
	readelf -a kernel

objdump:
	cargo objdump --target $(TARGET) -- -disassemble -print-imm-hex kernel

nm:
	cargo nm --target $(TARGET) -- kernel | sort