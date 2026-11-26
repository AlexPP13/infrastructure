.PHONY: jtbs build registry push ignition image usb vm vm-kargs vm-usb

OUTPUT_DIR := ./output
IMAGE ?= localhost/lenovo-e16-01:latest
LOCAL_REGISTRY ?= localhost:5000
DISK_DEVICE ?= /dev/sda
STREAM ?= next
PLATFORM ?= metal
ARCHITECTURE ?= x86_64
BOOT_IMAGE := $(OUTPUT_DIR)/boot.img
BOOT_SIZE ?= 4G

jtbs: image push

build:
	@podman build --pull=always --tag $(IMAGE) --file Containerfile filesystem
	@podman image tree $(IMAGE)

registry:
	@podman run -it --rm --replace --name lenovo-e16-01-registry \
		--publish 5000:5000 \
		docker.io/library/registry:2

push: build
	@podman tag $(IMAGE) $(VM_IMAGE)
	@podman push --tls-verify=false $(VM_IMAGE)

ignition:
# 	@rm -rf $(OUTPUT_DIR)
	@mkdir -p $(OUTPUT_DIR)

	@podman run --pull=always --interactive --rm \
		--workdir /workspace \
		--volume $(CURDIR)/bootstrap-secrets.yaml:/workspace/bootstrap-secrets.yaml:Z \
		--volume $(CURDIR)/filesystem/etc:/workspace/etc:Z \
		--volume $(CURDIR)/output:/workspace/output:Z \
		quay.io/coreos/butane:release \
		--pretty --strict --files-dir /workspace ./bootstrap-secrets.yaml \
		--output ./output/bootstrap-secrets.ign

	@podman run --interactive --rm \
		--workdir /workspace \
		--volume $(CURDIR)/bootstrap-dev.yaml:/workspace/bootstrap-dev.yaml:Z \
		--volume $(CURDIR)/output:/workspace/output:Z \
		quay.io/coreos/butane:release \
		--pretty --strict ./bootstrap-dev.yaml \
		--output ./output/bootstrap-dev.ign

	@podman run --interactive --rm \
		--workdir /workspace \
		--volume $(CURDIR)/bootstrap.yaml:/workspace/bootstrap.yaml:Z \
		--volume $(CURDIR)/filesystem/boot:/workspace/boot:Z \
		--volume $(CURDIR)/filesystem/etc:/workspace/etc:Z \
		--volume $(CURDIR)/filesystem/var:/workspace/var:Z \
		--volume $(CURDIR)/output/bootstrap-secrets.ign:/workspace/bootstrap-secrets.ign:Z \
		--volume $(CURDIR)/output/bootstrap-dev.ign:/workspace/bootstrap-dev.ign:Z \
		--volume $(CURDIR)/output:/workspace/output:Z \
		quay.io/coreos/butane:release \
		--pretty --strict --files-dir /workspace ./bootstrap.yaml \
		--output ./output/bootstrap.ign

image: ignition
	@rm -f $(BOOT_IMAGE)
	@truncate --size $(BOOT_SIZE) $(BOOT_IMAGE)
	@loop_device=$$(sudo losetup --find --show --partscan $(BOOT_IMAGE)); \
	trap 'sudo losetup --detach "$$loop_device"' EXIT; \
	sudo podman run --interactive --pull=always --rm --privileged \
		--volume /dev:/dev \
		--volume /run/udev:/run/udev \
		--volume $(CURDIR)/$(OUTPUT_DIR):/output:ro \
		quay.io/coreos/coreos-installer:release install \
		--stream $(STREAM) \
		--platform $(PLATFORM) \
		--architecture $(ARCHITECTURE) \
		--ignition-file /output/bootstrap.ign \
		"$$loop_device"

usb:
	test -f $(BOOT_IMAGE)
	test -b $(DISK_DEVICE)
	sudo dd if=$(BOOT_IMAGE) of=$(DISK_DEVICE) bs=4M conv=fsync status=progress

# Create an isolated FCOS test machine. Its first boot applies Ignition, stages
# the OCI rebase, and reboots into the configured image. SSH is forwarded to
# localhost:2222 after the rebase completes.
VM_IMAGE ?= $(LOCAL_REGISTRY)/lenovo-e16-01:latest
VM_DIR := $(OUTPUT_DIR)/vm
VM_NVME0_DISK := $(VM_DIR)/nvme0.img
VM_NVME1_DISK := $(VM_DIR)/nvme1.img
VM_NVME_SIZE0 ?= 100G
VM_NVME_SIZE1 ?= 80G
VM_OVMF_CODE ?= /usr/share/edk2/ovmf/OVMF_CODE.fd
VM_OVMF_VARS ?= /usr/share/edk2/ovmf/OVMF_VARS.fd
VM_QEMU_ACCEL ?= kvm
VM_QEMU_CPU ?= host
VM_SERIAL_KARG ?= console=ttyS0,115200n8
VM_BOND_NETWORK ?= 13.0.0.0/24
VM_BOND_GATEWAY ?= 13.0.0.1

# Copy the base image and append QEMU-specific kernel arguments.
vm-kargs:
	test -f $(BOOT_IMAGE)
# 	rm -rf $(VM_DIR)
	mkdir -p $(VM_DIR)
	truncate --size $(BOOT_SIZE) $(VM_DIR)/boot.img
	loop_device=$$(sudo losetup --find --show --partscan $(VM_DIR)/boot.img); \
	trap 'sudo losetup --detach "$$loop_device"' EXIT; \
	sudo podman run --interactive --pull=always --rm --privileged \
		--volume /dev:/dev \
		--volume /run/udev:/run/udev \
		--volume $(CURDIR)/$(OUTPUT_DIR):/output:ro \
		quay.io/coreos/coreos-installer:release install \
		--image-file /output/boot.img \
		--insecure \
		--append-karg $(VM_SERIAL_KARG) \
		"$$loop_device"

vm: vm-kargs
	command -v qemu-system-x86_64 >/dev/null
	test -r $(VM_OVMF_CODE)
	test -r $(VM_OVMF_VARS)
# 	truncate --size $(VM_NVME_SIZE0) $(VM_NVME0_DISK)
# 	truncate --size $(VM_NVME_SIZE1) $(VM_NVME1_DISK)
	cp $(VM_OVMF_VARS) $(VM_DIR)/OVMF_VARS.fd
	qemu-system-x86_64 \
		-machine q35,accel=$(VM_QEMU_ACCEL) \
		-cpu $(VM_QEMU_CPU) \
		-m 8192 \
		-smp 4 \
		-device pxb-pcie,id=pcie.5,bus=pcie.0,bus_nr=5,addr=0x5 \
		-device pcie-root-port,id=enp6s0,bus=pcie.5,chassis=6,slot=6,addr=0x1 \
		-device pcie-root-port,id=enp7s0,bus=pcie.5,chassis=7,slot=7,addr=0x2 \
		-drive if=pflash,format=raw,readonly=on,file=$(VM_OVMF_CODE) \
		-drive if=pflash,format=raw,file=$(VM_DIR)/OVMF_VARS.fd \
		-drive if=none,id=boot,file=$(VM_DIR)/boot.img,format=raw \
		-device virtio-blk-pci,drive=boot,serial=coreos-boot-disk,bus=pcie.0 \
		-drive if=none,id=nvme0,file=$(VM_NVME0_DISK),format=raw \
		-device nvme,drive=nvme0,serial=lenovo-e16-01-nvme0,bus=pcie.0 \
		-drive if=none,id=nvme1,file=$(VM_NVME1_DISK),format=raw \
		-device nvme,drive=nvme1,serial=lenovo-e16-01-nvme1,bus=pcie.0 \
		-vga none \
		-device virtio-vga,bus=pcie.0 \
		-device qemu-xhci,id=usb,bus=pcie.0 \
		-device usb-tablet,bus=usb.0 \
		-display gtk \
		-monitor none \
		-serial stdio \
		-netdev user,id=bond-enp6s0,net=$(VM_BOND_NETWORK),host=$(VM_BOND_GATEWAY) \
		-device virtio-net-pci,netdev=bond-enp6s0,bus=enp6s0,addr=0x0 \
		-netdev user,id=bond-enp7s0,net=$(VM_BOND_NETWORK),host=$(VM_BOND_GATEWAY),hostfwd=tcp::2222-:22 \
		-device virtio-net-pci,netdev=bond-enp7s0,bus=enp7s0,addr=0x0

# Boot the test VM from the FCOS installation written to the physical USB
# device. Run `make usb` first to write $(BOOT_IMAGE) to $(DISK_DEVICE).
vm-usb:
	command -v qemu-system-x86_64 >/dev/null
	test -r $(VM_OVMF_CODE)
	test -r $(VM_OVMF_VARS)
	test -b $(DISK_DEVICE)
	sudo test -r $(DISK_DEVICE) -a -w $(DISK_DEVICE)
# 	rm -rf $(VM_DIR)
	mkdir -p $(VM_DIR)
# 	truncate --size $(VM_NVME_SIZE0) $(VM_NVME0_DISK)
# 	truncate --size $(VM_NVME_SIZE1) $(VM_NVME1_DISK)
	cp $(VM_OVMF_VARS) $(VM_DIR)/OVMF_VARS.fd
	sudo qemu-system-x86_64 \
		-machine q35,accel=$(VM_QEMU_ACCEL) \
		-cpu $(VM_QEMU_CPU) \
		-m 8192 \
		-smp 4 \
		-device pxb-pcie,id=pcie.5,bus=pcie.0,bus_nr=5,addr=0x5 \
		-device pcie-root-port,id=enp6s0,bus=pcie.5,chassis=6,slot=6,addr=0x1 \
		-device pcie-root-port,id=enp7s0,bus=pcie.5,chassis=7,slot=7,addr=0x2 \
		-drive if=pflash,format=raw,readonly=on,file=$(VM_OVMF_CODE) \
		-drive if=pflash,format=raw,file=$(VM_DIR)/OVMF_VARS.fd \
		-drive if=none,id=usbboot,file=$(DISK_DEVICE),format=raw,cache=none,aio=native \
		-drive if=none,id=nvme0,file=$(VM_NVME0_DISK),format=raw \
		-device nvme,drive=nvme0,serial=lenovo-e16-01-nvme0,bus=pcie.0 \
		-drive if=none,id=nvme1,file=$(VM_NVME1_DISK),format=raw \
		-device nvme,drive=nvme1,serial=lenovo-e16-01-nvme1,bus=pcie.0 \
		-vga none \
		-device virtio-vga,bus=pcie.0 \
		-device qemu-xhci,id=usb,bus=pcie.0 \
		-device usb-storage,drive=usbboot,bus=usb.0,bootindex=1 \
		-device usb-tablet,bus=usb.0 \
		-display gtk \
		-monitor none \
		-serial stdio \
		-netdev user,id=bond-enp6s0,net=$(VM_BOND_NETWORK),host=$(VM_BOND_GATEWAY) \
		-device virtio-net-pci,netdev=bond-enp6s0,bus=enp6s0,addr=0x0 \
		-netdev user,id=bond-enp7s0,net=$(VM_BOND_NETWORK),host=$(VM_BOND_GATEWAY),hostfwd=tcp::2222-:22 \
		-device virtio-net-pci,netdev=bond-enp7s0,bus=enp7s0,addr=0x0
