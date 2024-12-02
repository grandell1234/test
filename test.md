# Architecture and toolchain
ARCH := arm
TARGET := debug
CROSS_COMPILE := arm-linux-gnueabihf-

LDFILE := config/linker.ld
LD := $(CROSS_COMPILE)gcc
GRUB_CFG := config/grub.cfg

# Path to compiled Rust library
LIB_PATH := target/$(ARCH)-oreneta/$(TARGET)/liboreneta.a

ASM_SRC_FILES := $(wildcard src/arch/$(ARCH)/asm/*.asm)
ASM_OBJ_FILES := $(patsubst src/arch/$(ARCH)/asm/%.asm, build/arch/$(ARCH)/asm/%.o, $(ASM_SRC_FILES))

build/oreneta.iso: build/kernel.bin
	@mkdir -p build/isofiles/boot/grub
	@cp build/kernel.bin build/isofiles/boot/
	@cp $(GRUB_CFG) build/isofiles/boot/grub
	@grub-mkrescue -o $@ build/isofiles
	@rm -r build/isofiles

run: build/oreneta.iso
	qemu-system-arm -M versatilepb -kernel $<

build/kernel.bin: rustbuild $(ASM_OBJ_FILES)
	$(LD) -T $(LDFILE) -o $@ -ffreestanding -nostdlib $(ASM_OBJ_FILES) $(LIB_PATH) -lgcc

rustbuild:
	cargo build

clean:
	rm -rf build
	cargo clean

# Assembly compilation rule
build/arch/$(ARCH)/asm/%.o: src/arch/$(ARCH)/asm/%.asm
	mkdir -p $(shell dirname $@)
	echo "[ASM] $<"
	$(CROSS_COMPILE)gcc -c -x assembler-with-cpp -o $@ $<