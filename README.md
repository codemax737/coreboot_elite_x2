# coreboot_elite_x2
Coreboot_elite_x2_1012_g1
Ai Generated Writing ( workign coreboot  HP Elite x2 1012 G1 " device verified "  )  NOTE , AI CAN MAKE MISTAKE 
[Success / How-To] Coreboot + LinuxBoot/Heads on the HP Elite x2 1012 G1 (Skylake-Y) — Full Thunderbolt 3, USB-C 3.1, DisplayPort Alt-Mode, Wi-Fi, & Clean S5 Shutdown
After extensive hardware reverse-engineering and register-level debugging, Coreboot (with both LinuxBoot/Heads and SeaBIOS payloads) is now running at 100% functionality on the HP Elite x2 1012 G1 detachable 2-in-1 tablet.
This post documents the hardware architecture, the root causes behind the tricky platform bugs (HP Sure Start SPI verification, Alpine Ridge Thunderbolt 3 PCIe link training, 32-bit SMM S5 shutdown reboots, and Nuvoton EC anti-bootloop protection), and a complete step-by-step guide to building and flashing the ROM from any stock 16 MB BIOS dump.
1. Hardware & Feature Matrix
Subsystem
Hardware Component
Status
Notes
CPU / SoC
Intel Core m5-6Y57 (Skylake-Y) + Sunrise Point-LP PCH
Working
Locked to 800 MHz LFM (under 1W) during boot; 2.2 GHz nominal at OS handoff
Memory
8 GB LPDDR3-1866 (Dual-Channel Soldered)
Working
Uses stock 256-byte SPD extracted from UEFI image
Security / Boot
HP Sure Start (Private 8 MB + Primary 16 MB SPI)
Working
Shrunk IFD BIOS region (0x00200000–0x00CFFFFF) preserves top 3 MB stock signature slice
Display (Internal)
12" 1920x1280 eDP (Intel HD Graphics 515)
Working
Hardware 180° panel rotation bits cleared in PLANE_CTL_1
Thunderbolt 3 / USB-C
Intel DSL6340 Alpine Ridge 2C (00:1c.0 → 01:00.0..15:00.0)
Working
Full PCIe tree, NHI (03:00.0), USB 3.1 SuperSpeed xHCI (15:00.0), hotplug & cold-plug
Type-C Display Out
TI TPS65982 PD + Intel South Display Hotplug
Working
HDMI / DisplayPort Alt-Mode via SHOTPLUG_CTL & SFUSE_STRAP
Wi-Fi & Bluetooth
Intel Wireless-AC 8260 (16:00.0) + BT (usb 1-9)
Working
Enabled via EC RAM 0xEA = 0x00; omit generic WF00 ACPI _PRR
Power Management
ACPI S0 / S3 / S5 (poweroff & reboot)
Working
Pure CPU I/O SMM S5 intercept + CMOS 0x72 warm-reboot bypass

2. The 5 Major Hardware Boss Fights (And How We Solved Them)
2.1 Bypassing HP Sure Start Without Touching the 8 MB Security Chip
The motherboard has two SPI flash chips:
Primary 16 MB SPI (W25Q128.V): Holds the IFD, Intel ME, BIOS region, and a 3 MB HP Sure Start signature/NVRAM block at the top of the chip (0x00D00000–0x00FFFFFF).
Private 8 MB SPI (W25Q64JV): Connected to the EC/Sure Start controller.
If you overwrite 0x00D00000–0x00FFFFFF on the 16 MB chip, the 8 MB chip detects a signature mismatch and refuses to release the board from reset.
Solution: We patched FLREG1 (BIOS region) in the Intel Flash Descriptor (ifd.bin offset 0x44) from 0x0fff0200 (14 MB) down to 0x0cff0200 (0x00200000–0x00CFFFFF, 11 MB), set CONFIG_ROM_SIZE=0x00d00000 (13 MB) and CONFIG_CBFS_SIZE=0x00b00000 (11 MB) in Coreboot, and appended the authentic 3 MB slice (hp_stock_top_3mb.bin) to the top of the 13 MB Coreboot image. You only ever flash the 16 MB chip (W25Q128.V).
2.2 Keeping the Fanless Skylake-Y CPU Cool During Boot
Because the Elite x2 1012 G1 is a thin fanless tablet, running FSP memory/silicon init and PCIe polling delays at full turbo frequency heats the CPU rapidly.
Solution: On the very first line of bootblock_mainboard_init(), we write 0x0800 to MSR_IA32_PERF_CTL (0x199), locking the CPU to 800 MHz Low Frequency Mode (under 1W) throughout bootblock, romstage, and ramstage. Right before OS handoff in mainboard_enable_backlight(), we restore MSR 0x199 to 0x1600 (2.2 GHz nominal).
2.3 Waking the Intel DSL6340 Alpine Ridge Thunderbolt 3 Controller
Getting 00:1c.0 (Root Port 1) to enumerate the Alpine Ridge switch (01:00.0 → 02:00.0..02:02.0 → 03:00.0 TB3 NHI and 15:00.0 USB 3.1 xHCI) required solving four hardware-level quirks:
Reference Clock & PLTRST# Synchronization (bootblock.c): On Sunrise Point-LP, PLTRST# (PERST#) deasserts on cold power-up before bootblock unmasks the 100 MHz PCIe reference clocks (PID_ICC 0xDC offset 0x1000 = 0x80999981) and drives GPP_E12 (TBT_FORCE_PWR) HIGH. Without a running REFCLK during the PLTRST# rising edge, Alpine Ridge's SerDes links up (LnkSta = 0x7042), but its internal PCIe Configuration Space responder wedges (VDID = 0xffffffff). We solved this with a one-time battery-backed CMOS 0x72 == 0xA5 warm reset (outb(0x06, 0xcf9)) in bootblock.c 400 ms after asserting ICC 0x1000 and GPP_E12 = 1. (Note: You must use 0x06 SYS_RST | RST_CPU and never 0x04, because 0x04 omits PLTRST# and leaves Cache-As-RAM No-Evict Mode active, hanging FSP-T TempRamInit).
Gen 3 Equalization Hang → 50 ms Link Disable Gen 2 Downshift: Alpine Ridge's NVM starts its upstream port at Gen 3 (8.0 GT/s). On 00:1c.0, initial link detection must be armed at Gen 3 (LCTL2 0x70 = 0x0003, LCTL 0x50 = 0x0c60) so the link reaches DLActive = 1 (LnkSta = 0x7043 or 0x7843). However, at Gen 3, equalization loops indefinitely (LinkTraining bit 11 stuck at 1). Once DLActive == 1 is reached, pulsing Link Disable for 50 ms with Target Speed = Gen 2 (0x70 = 0x0002, 0x50 = 0x0050 → 50 ms → 0x50 = 0x0040) triggers a clean Hot Reset downshift into Gen 2 x4 (LnkSta = 0x7042).
Dual VDID (0x00) & TB2P (0x548) Polling: After downshifting to Gen 2 (0x7042), Alpine Ridge's internal microcontroller takes 0.5s–2.2s to initialize. Crucially, its PCIe mailbox status register (TB2P at 01:00.0 offset 0x548, MMIO 0xE0100548) becomes readable (0x00000000 instead of 0xffffffff) before Vendor/Device ID (0xE0100000) stops returning 0xffffffff!
The 3-Command Mailbox Sequence (0x31 → 0x0D → 0x33): As soon as 0xE0100548 != 0xffffffff, writing COMMAND = 0x0007 to 0xE0100004 and dispatching 0x31 (BOOT_ON) → 0x0D (OS_UP) → 0x33 (USB_ON — must be sent last!) to P2TB (0xE010054C) causes 0xE0100000 to immediately flip to 0x15768086 and powers up downstream bridges 02:00.0..02:02.0 and 15:00.0 (8086:15b5 USB 3.1 SuperSpeed xHCI).
2.4 Fixing S5 Shutdown (poweroff) Reboots in 32-Bit SMM
Two bugs caused poweroff to reboot the tablet:
Coreboot's default Skylake SMM sleep handler (smihandler_southbridge_sleep) calls MMIO-based PCI config functions (0xE0000000+). In flat 32-bit SMM without identity page tables above 0xE0000000, any MMIO config access triggers an unhandled #PF page-fault triple-fault reset.
In the Sunrise Point-LP PM1_EN register (I/O port 0x1802), Bit 14 (0x4000) is PCIEXPWAK_DIS (PCI Express Wake DISABLE). Writing 0x0000 to PM1_EN enables PCIe wake from S5!
Solution: We placed a 100% pure CPU I/O-port ACPI_S5 intercept at line 1 of smihandler_southbridge_sleep() using only 0xCF8/0xCFC and 0x1800–0x189C: it clears PME_EN on XHCI/RP01/RP09, sets AFTERG3_EN = 1 (Bit 0 of GEN_PMCON_B at 00:1f.2 offset 0xA4), zeros GPE0_EN, writes 0x4000 (PCIEXPWAK_DIS = 1) to PM1_EN (0x1802), clears PM1_STS (0x1800 = 0xFFFF), and asserts SLP_TYP=5 | SLP_EN in PM1_CNT (0x1804).
2.5 Preventing reboot from Tripping the Nuvoton EC Anti-Bootloop Protection
When Linux executes a warm reboot, it issues Reset #1 (PLTRST#). If CMOS 0x72 is 0x00, bootblock.c issues Reset #2 (outb(0x06, 0xcf9)) 400 ms later. When the Nuvoton NPCE586 EC sees two PLTRST# drops within 400 ms, its anti-bootloop protection cuts the mainboard power rails (turning reboot into shutdown).
Solution: We pre-arm CMOS 0x72 = 0xA5 at the end of ramstage (mainboard_enable_backlight()) and via a lightweight systemd shutdown hook (hp-cmos-reboot), while clearing CMOS 0x72 = 0x00 only on S5 power-off. Thus, a cold boot from S5 runs the two-pass clock sync once, while warm reboots from Linux skip the second reset and reboot in a single pass.
3. Step-by-Step Guide to Building & Flashing
Step 1: Dump Your Stock 16 MB SPI Flash & Extract Blobs
Connect an external 3.3V SPI programmer (e.g., CH341A with a 3.3V mod or Raspberry Pi SPI) to the 16 MB Winbond W25Q128.V chip (do not touch the 8 MB W25Q64JV chip). Take two dumps and verify their SHA-256 hashes match:



Bash
flashrom -p ch341a_spi -c "W25Q128.V" -r stock_read1.bin
flashrom -p ch341a_spi -c "W25Q128.V" -r stock_read2.bin
sha256sum stock_read1.bin stock_read2.bin
cp stock_read1.bin hp_stock_full_16mb.bin


Extract the Flash Descriptor, Intel ME, and the critical top 3 MB HP Sure Start slice, then patch FLREG1 (0x44) and FLMSTR1 (0x80) in the IFD:



Bash
mkdir -p work/blobs
# 1. Extract 4 KB IFD, 2044 KB ME, and 3 MB Sure Start slice
dd if=hp_stock_full_16mb.bin of=work/blobs/ifd_stock.bin bs=4096 count=1
dd if=hp_stock_full_16mb.bin of=work/blobs/me.bin bs=4096 skip=1 count=511
dd if=hp_stock_full_16mb.bin of=work/blobs/hp_stock_top_3mb.bin bs=1M skip=13 count=3

# 2. Patch IFD: shrink BIOS region to 0x00200000-0x00CFFFFF (0x0cff0200) and unlock FLMSTR1
python3 -c '
with open("work/blobs/ifd_stock.bin", "rb") as f:
    ifd = bytearray(f.read())
ifd[0x44:0x48] = (0x0cff0200).to_bytes(4, "little") # FLREG1: 0x00200000 - 0x00CFFFFF
ifd[0x80:0x84] = (0xffffffff).to_bytes(4, "little") # FLMSTR1: unlock R/W
with open("work/blobs/ifd_shrunk.bin", "wb") as f:
    f.write(ifd)
'


(Also extract the 256-byte LPDDR3 spd.bin and Intel vbt.bin from hp_stock_full_16mb.bin using ifdtool -x and UEFIExtract).
Step 2: bootblock.c Implementation
In src/mainboard/hp/elite_x2_1012_g1/bootblock.c:



C
#include <arch/io.h>
#include <bootblock_common.h>
#include <cpu/x86/msr.h>
#include <delay.h>
#include <device/pnp_ops.h>
#include <ec/acpi/ec.h>
#include <gpio.h>
#include <pc80/mc146818rtc.h>
#include <soc/iomap.h>
#include <soc/pcr_ids.h>
#include <intelblocks/pcr.h>
#include "gpio.h"

void bootblock_mainboard_early_init(void)
{
    mainboard_configure_early_gpios();
    gpio_set(GPP_E12, 1); /* Assert TBT_FORCE_PWR early */
}

void bootblock_mainboard_init(void)
{
    /* 1. Lock CPU to 800 MHz LFM (<1W) immediately for cool boot */
    msr_t perf_ctl = { .lo = 0x0800, .hi = 0 };
    wrmsr(0x199, perf_ctl);

    /* 2. Enable Nuvoton EC ACPI mode and Wi-Fi/WWAN power rails */
    ec_set_ports(0x66, 0x62);
    ec_write(0x82, ec_read(0x82) | 0x04);
    ec_write(0xea, 0x00);

    /* 3. Unmask all PCH ICC PCIe reference clocks and assert GPP_E12 */
    pcr_write32(PID_ICC, 0x1000, 0x80999981);
    gpio_set(GPP_E12, 1);

    /* 4. One-time cold-boot PLTRST# reference clock synchronization */
    if (cmos_read(0x72) != 0xA5) {
        cmos_write(0xA5, 0x72);
        mdelay(400);
        outb(0x06, 0xcf9); /* Must be 0x06 (SYS_RST | RST_CPU) to cycle PLTRST# and NEM */
        while (1)
            asm volatile("hlt");
    } else {
        cmos_write(0x00, 0x72);
    }
}


Step 3: mainboard.c — Alpine Ridge TB3 Handshake & Display Init
In src/mainboard/hp/elite_x2_1012_g1/mainboard.c:



C
static void send_ar_mailbox_cmd(uintptr_t ar_ecam, uint32_t cmd)
{
    write32((void *)(ar_ecam + 0x54c), cmd);
    for (int i = 0; i < 50; i++) {
        uint32_t tb2p = read32((void *)(ar_ecam + 0x548));
        if (tb2p & 1)
            break;
        mdelay(10);
    }
    write32((void *)(ar_ecam + 0x54c), 0);
    mdelay(20);
}

void mainboard_silicon_init_params(FSPS_UPD *params)
{
    msr_t perf_ctl = { .lo = 0x0800, .hi = 0 };
    wrmsr(0x199, perf_ctl);

    gpio_set(GPP_E12, 1);
    pcr_write32(PID_ICC, 0x1000, 0x80999981);
    mdelay(1800); /* 1.8s at <1W 800 MHz for Alpine Ridge NVM & TI PD load */

    /* RP01 (00:1c.0): Alpine Ridge TB3 x4 - Keep Speed=0 (Auto/Gen3) for initial wake */
    params->PcieRpEnable[0] = 1;
    params->PcieRpClkReqSupport[0] = 0;
    params->PcieRpClkReqDetect[0] = 0;
    params->PcieRpHotPlug[0] = 1;
    params->PcieRpPcieSpeed[0] = 0;

    /* RP09 (00:1d.0): Intel 8260 Wi-Fi - Disable ASPM */
    params->PcieRpEnable[8] = 1;
    params->PcieRpClkReqSupport[8] = 0;
    params->PcieRpClkReqDetect[8] = 0;
    params->PcieRpAspm[8] = 0;
}

void mainboard_prepare_pcie(void)
{
    pci_devfn_t rp01 = PCI_DEV(0, 0x1c, 0);
    uintptr_t ar_ecam = 0xe0100000;

    pci_write_config32(rp01, 0x208, 0);      /* Clear L1SS_CTL1 */
    pci_write_config16(rp01, 0x58, 0x0000);  /* Mask SLTCTL IRQs during init */
    pci_write_config16(rp01, 0x5a, 0x01ff);  /* Clear SltSta */
    pci_write_config16(rp01, 0x50, 0x0040);  /* Common Clock */

    uint16_t lnksta1 = pci_read_config16(rp01, 0x52);
    if (!(lnksta1 & (1 << 13))) {
        pci_write_config16(rp01, 0x70, 0x0003); /* Gen 3 wake */
        pci_write_config16(rp01, 0x50, 0x0c60);
        for (int i = 0; i < 200; i++) {
            mdelay(10);
            lnksta1 = pci_read_config16(rp01, 0x52);
            if (lnksta1 & (1 << 13))
                break;
        }
    }

    /* ONLY pulse Link Disable when DLActive == 1 (0x7043 / 0x7843) */
    if (lnksta1 & (1 << 13)) {
        pci_write_config16(rp01, 0x70, 0x0002); /* Target Speed = Gen 2 */
        pci_write_config16(rp01, 0x50, 0x0050); /* Link Disable = 1 */
        mdelay(50);
        pci_write_config16(rp01, 0x50, 0x0040); /* Link Disable = 0 */

        for (int i = 0; i < 50; i++) {
            mdelay(20);
            lnksta1 = pci_read_config16(rp01, 0x52);
            if (lnksta1 & (1 << 13))
                break;
        }
        pci_write_config16(rp01, 0x5a, 0x01ff);
        pci_write_config32(rp01, 0x18, 0x00150100);
        pci_write_config16(rp01, PCI_COMMAND, 0x0007);

        mdelay(500); /* Trhfa post-downshift recovery */

        /* Poll BOTH VDID (0x00) and TB2P (0x548) up to 2,500 ms */
        for (int i = 0; i < 125; i++) {
            uint32_t vdid = read32((void *)(ar_ecam + 0x00));
            uint32_t tb2p = read32((void *)(ar_ecam + 0x548));
            if (vdid == 0x15768086 || tb2p != 0xffffffff) {
                write16((void *)(ar_ecam + 0x04), 0x0007);
                send_ar_mailbox_cmd(ar_ecam, 0x31); /* BOOT_ON */
                send_ar_mailbox_cmd(ar_ecam, 0x0d); /* OS_UP */
                send_ar_mailbox_cmd(ar_ecam, 0x33); /* USB_ON (MUST BE LAST) */
                mdelay(500);
                break;
            }
            mdelay(20);
        }
        pci_write_config16(rp01, 0x5a, 0x01ff);
        pci_write_config32(rp01, 0x18, 0x00000000);
    } else {
        /* Leave Gen 3 wake armed and enable hotplug interrupts for OS pciehp */
        pci_write_config16(rp01, 0x58, 0x0128);
    }
}


And in mainboard_enable_backlight() (BS_DEV_ENABLE):



C
/* Clear PLANE_CTL_1 180-degree rotation bits & enable South Display Hotplug for Type-C HDMI */
uint32_t plane_ctl = read32((void *)(gttmmadr + 0x70180));
write32((void *)(gttmmadr + 0x70180), plane_ctl & ~0x3);
write32((void *)(gttmmadr + 0xc4030), read32((void *)(gttmmadr + 0xc4030)) | 0x00100100);
write32((void *)(gttmmadr + 0xc2014), read32((void *)(gttmmadr + 0xc2014)) | 0x00000006);

/* Restore 2.2 GHz nominal CPU clock, enable SLP_SMI_EN, and pre-arm CMOS 0x72 for warm reboot */
msr_t perf_ctl = { .lo = 0x1600, .hi = 0 };
wrmsr(0x199, perf_ctl);
outl(inl(0x1830) | (1 << 4), 0x1830);
cmos_write(0xA5, 0x72);


Step 4: Pure CPU I/O-Port SMM S5 Shutdown Handler
At the very top (Line 1) of smihandler_southbridge_sleep() in src/soc/intel/common/block/smm/smihandler.c:



C
if (slp_typ == ACPI_S5) {
    /* 1. Clear CMOS 0x72 so next cold boot from S5 runs Pass A clock sync */
    outb(0x72, 0x72); outb(0x00, 0x73);
    outb(0x72, 0x70); outb(0x00, 0x71);

    /* 2. Clear PME_EN (bit 8) and PME_STS (bit 15) on XHCI, RP01, RP09 via I/O 0xCF8/0xCFC */
    outl(0x8000a074, 0xcf8); outl((inl(0xcfc) & ~(1 << 8)) | (1 << 15), 0xcfc);
    outl(0x8000e060, 0xcf8); outl((inl(0xcfc) & ~(1 << 8)) | (1 << 15), 0xcfc);
    outl(0x8000e0a4, 0xcf8); outl((inl(0xcfc) & ~(1 << 8)) | (1 << 15), 0xcfc);
    outl(0x8000e058, 0xcf8); outw(0x0000, 0xcfc);
    outl(0x8000e860, 0xcf8); outl((inl(0xcfc) & ~(1 << 8)) | (1 << 15), 0xcfc);
    outl(0x8000e8a4, 0xcf8); outl((inl(0xcfc) & ~(1 << 8)) | (1 << 15), 0xcfc);

    /* 3. Set AFTERG3_EN = 1 (Bit 0) in PMC GEN_PMCON_B (00:1f.2 0xA4) on S5 entry */
    outl(0x8000faa4, 0xcf8);
    outl(inl(0xcfc) | (1 << 0), 0xcfc);

    /* 4. Disable all GPE0 wake enables and clear GPE0 status */
    outl(0x00000000, ACPI_BASE_ADDRESS + 0x90);
    outl(0x00000000, ACPI_BASE_ADDRESS + 0x94);
    outl(0x00000000, ACPI_BASE_ADDRESS + 0x98);
    outl(0x00000000, ACPI_BASE_ADDRESS + 0x9c);
    outl(0xffffffff, ACPI_BASE_ADDRESS + 0x80);
    outl(0xffffffff, ACPI_BASE_ADDRESS + 0x84);
    outl(0xffffffff, ACPI_BASE_ADDRESS + 0x88);
    outl(0xffffffff, ACPI_BASE_ADDRESS + 0x8c);

    /* 5. Set PCIEXPWAK_DIS (Bit 14 = 0x4000) in PM1_EN (0x1802) and clear PM1_STS (0x1800) */
    outw(0x4000, ACPI_BASE_ADDRESS + 0x02);
    outw(0xffff, ACPI_BASE_ADDRESS + 0x00);

    /* 6. Clear SLP_SMI_EN and assert SLP_TYP=5 | SLP_EN in PM1_CNT (0x1804) */
    outl(inl(ACPI_BASE_ADDRESS + 0x30) & ~(1 << 4), ACPI_BASE_ADDRESS + 0x30);
    outl((5 << 10) | (1 << 13), ACPI_BASE_ADDRESS + 0x04);
    while (1)
        __asm__ __volatile__("hlt");
}


Step 5: Linux Kernel Parameters & Cold-Plug Helper Daemon (hp-tbt-wake.service)
Add the following parameters to your Linux kernel command line (/etc/default/grub or LinuxBoot /init):



Plaintext
pci=hpbussize=0x10,hpmemsize=64M usbcore.autosuspend=-1 iwlwifi.power_save=0 iwlmvm.power_scheme=1 modprobe.blacklist=hp_wmi


For cold boots where a passive Type-C hub was already plugged in before pressing the power button (leaving 00:1c.0 at LnkSta = 0x1801, DLActive = 0), install /usr/local/sbin/hp-tbt-wake.py as a systemd service (hp-tbt-wake.service). If 15:00.0 (8086:15b5) is absent and DLActive == 0 for 5 seconds after boot, it performs a single-shot software re-plug:
Masks SLTCTL (0x58 = 0x0000), sets LCTL (0x50 = 0x0050 Link Disable), and asserts Secondary Bus Reset (BCTL 0x3E |= 0x0040).
Pulses GPP_E12 (0xFDAE05E0) LOW (0x44000200) for 400 ms, then HIGH (0x44000201), and waits 1,500 ms.
Deasserts Secondary Bus Reset (BCTL 0x3E &= ~0x0040), arms Gen 3 wake (0x70 = 0x0003, 0x50 = 0x0c60), downshifts to Gen 2 (0x7042) upon DLActive == 1, sends 0x31 → 0x0D → 0x33 to 0xE010054C, restores SLTCTL = 0x0128, and triggers /sys/bus/pci/rescan.
Step 6: Assemble the 16 MB ROM, Verify Top 3 MB Integrity, & Flash
Compile the 13 MB Coreboot image (build/coreboot.rom) and append the 3 MB stock HP Sure Start slice:



Bash
cat build/coreboot.rom work/blobs/hp_stock_top_3mb.bin > coreboot_16mb_final.bin

# Verify exact 16,777,216-byte size and 100% bit-for-bit top 3 MB match:
dd if=coreboot_16mb_final.bin bs=1M skip=13 count=3 | sha256sum
sha256sum work/blobs/hp_stock_top_3mb.bin

# Flash ONLY to the 16 MB Winbond W25Q128.V SPI chip:
flashrom -p ch341a_spi -c "W25Q128.V" -w coreboot_16mb_final.bin


Upon first power-on after flashing, bootblock.c executes the 400 ms clock-sync reset at 800 MHz and boots directly into your payload with full Thunderbolt 3, USB 3.1 SuperSpeed, Type-C HDMI, Wi-Fi, Bluetooth, and clean ACPI power management.
