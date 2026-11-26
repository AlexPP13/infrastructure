# lenovo-e16-01

`Containerfile` defines the immutable host operating system. It replaces the
old first-boot package layering and virtio-win repository setup.

`bootstrap.yaml` remains the Butane source for machine-specific Ignition:
storage, users, kernel arguments, networking, firewall, libvirt, and Quadlet
configuration.

## Application sources

GNOME applications are installed as Flathub stable Flatpaks rather than
added to the host image as RPMs. This includes Calculator, Calendar,
Characters, Connections, Contacts, Decibels, Web (Epiphany), Evince, Logs,
Maps, Showtime, Camera (Snapshot), Weather, Disk Usage Analyzer (Baobab),
Fonts, and gThumb. Their multimedia support comes from their Flatpak runtimes
and extensions, independently of Fedora's host codec packages. The image
removes these RPM applications if they are included in the Silverblue base.

Terra supplies Mesa and Starship; RPM Fusion supplies FFmpeg and additional
GStreamer codecs. Terra's multimedia repository remains disabled. Other
desktop applications remain on Flathub stable.

The base is `fedora-silverblue:latest`. Image builds must validate third-party
repository compatibility. Flatpak applications remain on Flathub stable
independently of the host release.

### Flatpaks

System Flatpaks are declared in
`filesystem/etc/flatpak/preinstall.d/default.preinstall`. After the initial
rebase has booted into the OCI deployment, `flatpak-preinstall.service`
runs Flatpak's official `flatpak preinstall` mechanism once; Flathub is
configured in the image using Flatpak's static OS-vendor remote configuration.
The apps are downloaded into mutable `/var/lib/flatpak`, rather than being
included in the OCI image. Add or remove `[Flatpak Preinstall APP_ID]`
sections before generating a new Ignition configuration to change the
workstation defaults.

The installation runs as root, so it cannot use `core`'s session bus: that
bus rejects the root process during D-Bus credential authentication. The
service instead starts a temporary root-owned session bus with
`dbus-run-session`; Flatpak still uses the system helper for the system-wide
installation.

Users can opt out of an OS-declared app by uninstalling it; Flatpak records
that choice and does not automatically reinstall it.

### Existing installations

Ignition supplies the preinstall file only during provisioning, and the
preinstall service has a once-only marker. An image update alone does not
install the restored Flatpaks on an existing machine. From the repository
root on the host, copy the updated declarations and run preinstall explicitly:

```sh
sudo install -m 0644 lenovo-e16-01/filesystem/etc/flatpak/preinstall.d/default.preinstall \
    /etc/flatpak/preinstall.d/default.preinstall
sudo flatpak remote-modify --system --collection-id=org.flathub.Stable flathub
sudo dbus-run-session -- flatpak preinstall --system --noninteractive --assumeyes
```

Prefer installing and checking the Flatpaks before booting the image that
removes their RPM counterparts. Apps previously uninstalled by the user may
need an explicit `flatpak install --system flathub APP_ID` because preinstall
respects opt-outs. Existing Flatpak data in `~/.var/app/APP_ID` is retained;
settings and data created by the RPM applications are not automatically
migrated into the Flatpak sandbox. Preserve or export that data as needed.

On its first boot, Fedora CoreOS applies that Ignition configuration and
`rebase.service` stages the OCI image as the next deployment. The
service writes `/etc/lenovo-e16-01/rebased` before rebooting, so it runs only
once.

## Commands

Build locally with `make build`. It uses `--pull=always`, so the mutable
`fedora-silverblue:latest` base is refreshed rather than reused from Podman's
local cache. Set `IMAGE` to the GitHub Container Registry reference before
publishing, for example:

```sh
make IMAGE=ghcr.io/example/lenovo-e16-01:latest push
```

On every push to `main` that changes `lenovo-e16-01/`, GitHub Actions builds and
publishes `ghcr.io/<repository-owner>/lenovo-e16-01:latest`. The workflow can also
be run manually from the Actions tab.

## Operating-system updates

GNOME Software does not discover updates by polling the OCI tag tracked by a
bootc deployment. Ignition enables `bootc-fetch-apply-updates.timer`; it
periodically pulls the current deployment's image (for example,
`ghcr.io/alexpp13/lenovo-e16-01:latest`) and stages it for the next reboot.

After rebasing an already-installed machine to an image that predates this
change, enable the timer once:

```sh
sudo systemctl enable --now bootc-fetch-apply-updates.timer
```

To fetch, stage, and reboot into the latest GHCR image immediately, run:

```sh
sudo bootc upgrade --apply
```

Check the tracked image and staged deployment with `bootc status`.

Generate Ignition with `make ignition`.
The target requires the Git-ignored `bootstrap-secrets.yaml`, which contains
the `core` password hash and LUKS key material. It compiles that source into a
temporary Ignition fragment and merges it into the Git-ignored output.

## Installation

`rebase.service` targets the local QEMU test registry at `13.0.0.1:5000`.
Change its `ExecStart` image reference to the published GHCR image before a
physical installation, and remove
`filesystem/etc/containers/registries.conf.d/lenovo-e16-01-vm.conf` from `bootstrap.yaml`
when the registry uses HTTPS.

## ITE hardware monitoring

`bootstrap.yaml` creates `/etc/modules-load.d/it87.conf` through Ignition to
load Fedora's in-tree `it87` hardware-monitoring driver at boot. The driver
is already included in the workstation image. No out-of-tree module,
forced device IDs, or ACPI resource-conflict overrides are configured.

After provisioning, verify it on the physical workstation:

```sh
lsmod | grep '^it87'
journalctl -b -u systemd-modules-load.service
sudo journalctl -k -b --grep='it87|ACPI.*conflict'
```

Ignition runs only on first boot. On an existing host, enable it manually:

```sh
sudo install -d -m 0755 /etc/modules-load.d
echo it87 | sudo tee /etc/modules-load.d/it87.conf
sudo chmod 0644 /etc/modules-load.d/it87.conf
sudo modprobe it87
```

The driver needs supported hardware; loading it in a container or a VM
cannot validate detection of the workstation's sensor chip. Investigate
any unsupported-chip or ACPI conflict errors before adding module options.

## AMD VFIO configuration

The host enables AMD IOMMU (`amd_iommu=on`) with identity mappings for host
devices (`iommu=pt`). PCI devices are deliberately **not** bound to `vfio-pci`
by vendor/device ID at boot. Both domain definitions use `managed="yes"`, so
libvirt detaches each configured host device from its normal driver when the
guest starts and reattaches it when the guest stops. This lets the AMD GPU and
USB controllers remain usable by the host while no VM is running.

The first-boot `hardware-initramfs.service` embeds the VFIO drivers in the
initramfs, ensuring they are available before libvirt starts. It does not bind
any device. Do not add `vfio-pci.ids`, `rd.driver.pre=vfio-pci`, or
`pcie_acs_override` unless you intentionally require permanent early device
binding. The latter does not provide hardware IOMMU isolation.

Before using a device in a guest, verify that every device in its IOMMU group
is safe to pass through together:

```sh
find /sys/kernel/iommu_groups/ -type l
virsh nodedev-dumpxml pci_0000_0d_00_0
```

Update the `<hostdev>` source BDFs in the domain XML to match the actual
machine. The existing BDFs are hardware-specific examples and must not be
copied blindly. A GPU normally needs both its display and HDMI/DP-audio
functions; a USB controller needs its whole IOMMU group.

`amd_pstate=active` requests the AMD hardware-managed CPU performance driver.
The guests already use `host-passthrough`, cache passthrough, AMD `topoext`,
and disable migration, which exposes the CPU features KVM can safely virtualize.
`kvm_amd.nested=1` is retained only for nested virtualization. `mitigations=off`
is a security/performance trade-off: remove it on a host that runs untrusted
workloads.

Write a Fedora CoreOS installation to the selected USB or disk with:

```sh
make DISK_DEVICE=/dev/sdX usb
```

This command erases `DISK_DEVICE`. The installed FCOS system uses Ignition on
its first boot, rebases to the OCI image, and reboots into that deployment.

## Rebase Test VM

Install `qemu-system-x86` and the OVMF firmware package, then run:

```sh
make vm
```

Uncomment the `bootstrap-dev.ign` merge in `bootstrap.yaml` before building a
local test image, then run `make image` and `make push` before launching the
VM. The target copies the 16 GiB FCOS boot image into `output/vm/` and creates
two 80 GiB NVMe disks.
The current test configuration creates 64 GiB RAID root members and a 32 GiB
RAID0 `/var`. Ignition clears the USB down to a 128 MiB ESP and uses the
remaining space for `/boot`. Ignition creates a mount unit only for `/boot`
before the OCI rebase runs and reboots. It uses a static management interface
on `13.0.0.20` and a DHCP LAN bridge, launches a GTK window with virtio
graphics and USB tablet input, forwards guest SSH to `localhost:2222`, and
retains serial logs in the launching terminal.

The default uses KVM. On a machine without KVM, use `VM_QEMU_ACCEL=tcg VM_QEMU_CPU=max`.

To boot the VM from the physical USB installed at `/dev/sda` instead of the
copied boot image, write the USB and launch it with:

```sh
make usb
make vm-usb
```

`vm-usb` attaches `DISK_DEVICE` as a USB storage device and requires sudo to
read and write the physical disk. Override it for a differently named USB
device, for example `make DISK_DEVICE=/dev/sdX vm-usb`.

### Iterating on a running VM

After changing `Containerfile`, publish a replacement deployment without
recreating the VM disks:

```sh
make push
```

From inside the VM, stage that image and reboot into it:

```sh
sudo rpm-ostree rebase ostree-unverified-registry:13.0.0.1:5000/lenovo-e16-01:latest
sudo systemctl reboot
```
