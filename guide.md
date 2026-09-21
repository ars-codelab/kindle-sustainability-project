# Kindle Paperwhite 2 (6th Gen) Automation & Jailbreak Manual

This specification is designed for an autonomous local agent running on macOS to stage packages, execute exploits safely, perform system backups, and deploy an unbloated KOReader environment.

---

## Technical Specifications & Architecture Overview

* **Target Hardware:** Kindle Paperwhite 2 (Model `EY21`, late 2013–2014)
* **Processor:** Freescale/NXP i.MX6 SoloLite (1 GHz Cortex-A9)
* **RAM:** 256 MB or 512 MB (tight memory constraints; stock Java stack utilizes ~70–85%)
* **Assumed Target Firmware:** `5.12.2.2` (the final official release for PW2)
* **Exploit Vector:** **WatchThis** software exploit (via demo mode packaging)
* **Primary Target:** KOReader in **No-Framework mode** (Amazon Java UI suspended)

---

## Agent Operational Constraints

1. **Volume Mount:** Kindle mass storage mounts at `/Volumes/Kindle` on macOS.
2. **Read-Only / Human Boundary:** The agent cannot tap the physical e-ink screen. When on-device input is required, the agent must pause, output clear on-screen instructions, and await human confirmation before proceeding.
3. **Fail-Safe Checkpoints:** If the mount point is absent, or if directory contents are corrupted, halt immediately.

---

1. **Pre-Flight Checks and Complete Flash Storage Backup:** 10-15 min.
The agent must verify volume availability and take a bit-for-bit or file-level backup of the entire user-accessible storage partition before touching any filesystem structures.

1. **Detect Volume:** Ensure `/Volumes/Kindle` is present.
2. **Execute Local Backup:**
Create a local backup directory on macOS and sync the complete Kindle drive:

```bash
BACKUP_DIR="$HOME/Kindle_PW2_Backup_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"
rsync -aHAX --progress /Volumes/Kindle/ "$BACKUP_DIR/user_storage/"

```

3. **Record Device Metadata:** Read and save any metadata files (e.g., `system/` attributes if visible) to the backup folder.
4. **Verification:** Confirm that `$BACKUP_DIR/user_storage/` contains all existing directories (`documents`, etc.) and matches the source size via `du -sh`.


2. **Disable OTA Updates and Block Telemetry:** 2 min.
Prevent Amazon background tasks from pulling over-the-air updates or overwriting root binaries.

1. Create an immutable directory in place of the update staging file:

```bash
mkdir -p /Volumes/Kindle/update.bin.tmp.partial
chmod 000 /Volumes/Kindle/update.bin.tmp.partial

```

2. **Verification:** Test that the file cannot be written to or replaced by running `touch /Volumes/Kindle/update.bin.tmp.partial/test` and confirming it returns `Permission denied`.


3. **Stage the WatchThis Exploit Packages:** 5 min.
Fetch and place the exploit payload onto the device storage.

1. Obtain the official **WatchThis** distribution archive (from the MobileRead Kindle Developer's Corner).
2. Create the hidden demo staging directory on the Kindle:

```bash
mkdir -p /Volumes/Kindle/.demo

```

3. Copy the exploit payload files into `/Volumes/Kindle/` and `/Volumes/Kindle/.demo/` exactly as specified in the WatchThis distribution tree.
4. Run macOS cleanup to remove stray AppleDouble files:

```bash
dot_clean /Volumes/Kindle
diskutil eject /Volumes/Kindle

```

5. **Verification:** `diskutil list` confirms `/Volumes/Kindle` is cleanly unmounted.


4. **Execute On-Device Jailbreak (Human Action Required):** 5 min.
The agent halts here and prompts the human user to trigger the on-device sequence.

1. Unplug the USB cable from the Kindle.
2. Put the device in **Airplane Mode** via the Quick Actions / Settings menu.
3. Open the search bar on the Kindle home screen, type:

```text
;enter_demo

```

and tap Enter.
4. Reboot the device when prompted or via the power button.
5. In Demo mode, tap through the initial setup prompt, complete the bypass gesture (double-tap with two fingers or the specific WatchThis trigger pattern), and open the staged demo document.
6. The exploit executes, displays the jailbreak text overlay, and reboots the Kindle into an open-root state.


5. **Stage MRPI, Hotfix, KUAL, and KOReader:** 5 min.
Once the device reboots, connect it back to the Mac via USB. The agent resumes file staging.

1. Wait for `/Volumes/Kindle` to mount.
2. Deploy the core utility packages:
* Download and extract **MRPI** (MobileRead Package Installer) into `/Volumes/Kindle/` (creates `/Volumes/Kindle/mrpackages` and `/Volumes/Kindle/extensions`).
* Download the latest **JailBreak Hotfix** (`Update_jailbreak_hotfix_*.bin`) and place it inside `/Volumes/Kindle/mrpackages/`.
* Download **KUAL (coplate booklet version)** `.bin` file and place it inside `/Volumes/Kindle/mrpackages/`.
* Download the **KOReader Kindle package** (`koreader-kindle-*.zip`). Extract its contents so that `koreader` and `extensions` sit directly on `/Volumes/Kindle/`.


3. Create a dedicated folder for personal EPUBs:

```bash
mkdir -p /Volumes/Kindle/books

```

4. Clean macOS artifacts and unmount:

```bash
dot_clean /Volumes/Kindle
diskutil eject /Volumes/Kindle

```

5. **Verification:** Verify `/Volumes/Kindle/mrpackages/` contains the hotfix `.bin` and KUAL `.bin` before unmounting.


6. **Execute Package Installation via MRPI (Human Step):** 3 min.
The agent signals the human to install the staged packages into root.

1. Disconnect the USB cable.
2. In the Kindle search bar, type:

```text
;log mrpi

```

and tap Enter.
3. The screen will invoke the MobileRead Package Installer, display terminal output on the e-ink screen, patch the rootfs, and reboot.
4. **Verification:** Upon boot, a new "KUAL" document appears in your library list.


7. **Configure De-Bloating & KOReader Autostart:** 5 min.
To eliminate Amazon memory bloat and launch directly into KOReader:

1. Connect the Kindle back to your Mac via USB.
2. Navigate to `/Volumes/Kindle/koreader`.
3. In KOReader’s extension configuration or through KUAL:
* Ensure KOReader is set to use the **"No Framework"** wrapper (`koreader.sh` executes `stop lab126_gui` on start and `start lab126_gui` on exit).


4. *(Optional Autostart)* If setting up an Upstart init script (`/etc/upstart/koreader.conf`), implement a safety-latch check:

```bash
# Require a flag file on user storage to avoid bootlooping if KOReader corrupts
if [ -f /mnt/us/ENABLE_KOREADER_AUTOSTART ]; then
    stop lab126_gui
    /mnt/us/koreader/koreader.sh
fi

```

Touch the safety file to enable:

```bash
touch /Volumes/Kindle/ENABLE_KOREADER_AUTOSTART

```

5. Eject cleanly:

```bash
dot_clean /Volumes/Kindle
diskutil eject /Volumes/Kindle

```

6. **Verification:** Launching KOReader turns the screen black for a moment as the Java VM dies; memory consumption drops from ~80% to under 30%.


---

## Wi-Fi EPUB Transfer Configuration (Post-Install)

Once inside KOReader, no USB connection is required to load books:

1. Tap the top menu bar in KOReader and navigate to the **Network** (Wi-Fi icon) tab.
2. Turn on **Local HTTP Server (WebDAV / HTTP Transfer)**.
3. An IP and port will display on the screen (e.g., `[http://192.168.1.45:8080](http://192.168.1.45:8080)`).
4. On your Mac, open any browser and enter the address. Drag and drop `.epub` files directly into the `/mnt/us/books` folder.
