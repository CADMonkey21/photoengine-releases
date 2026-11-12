# PhotoEngine Studio

**PhotoEngine Studio** is a high-performance standalone and live AutoCAD photometric calculation engine for exterior site lighting, 3D interior room radiosity, life-safety emergency egress, and automated CAD schedule generation.

## Key Capabilities

- **Fast Multi-Core Photometric Solver:** Point-by-point direct illuminance, 3D interior room radiosity surface bounce (ceiling/wall/floor reflectances), property-line horizontal/vertical spill, and 3D shadow-casting obstructions.
- **Dual Normal & Emergency Egress Passes:** Simultaneous calculation of Normal power and Life-Safety Emergency battery/inverter egress layers (`E-LITE-CALC-PTS` and `E-LITE-CALC-EM-PTS`).
- **Plottable Contours & Live Aiming Templates:** Clipped ISO-footcandle contour polylines with inline labels (`E-LITE-CNTR`) and non-plotting single-fixture throw rings (`E-LITE-TMPL`) that move and rotate with your CAD blocks.
- **Live Windows COM AutoCAD Bridge:** Runs side-by-side with AutoCAD without modal dialog lockups—stamp blocks, draw zones, toggle layer visibility, and place customizable schedules in one click.
- **Two-Tier Portable IES Library:**
  - **Tier 1 — Portable Favorites Sidecar (`IES_Favorites.zip`):** ~15,000+ curated fixtures with an embedded TSV index that loads in milliseconds with zero loose `.ies` files on disk.
  - **Tier 2 — Full Reference Vault (`IES_Vault` / `IES_Staging_Zips`):** 680,000+ manufacturer fixtures indexed into a ~19 MB `.pe_vault_index.tsv.gz` file and read directly from raw brand `.zip` archives on demand without extracting files to NTFS.

## Workstation Deployment Guide

### 1. Choose Your Deployment Footprint

Create a permanent folder on the target Windows PC (for example, `C:\Tools\PhotoEngine\`) and copy:

| File / Folder | Required? | Purpose |
| :--- | :--- | :--- |
| `PhotoEngine.exe` | **Required** | Standalone Wails WebView2 Studio + Solver + CLI (~12 MB) |
| `IES_Favorites.zip` | **Recommended** | Tier 1 curated IES library sidecar (~38 MB). Place next to `PhotoEngine.exe`. |
| `IES_Vault\` *(or `IES_Staging_Zips\`)* | **Optional** | Brand `.zip` archives + `.pe_vault_index.tsv.gz` (~1.5 GB). Local or shared NAS folder. |

To keep local installs under 55 MB across multiple machines, host `IES_Vault\` (containing the brand `.zip` files and `.pe_vault_index.tsv.gz`) on a shared server or NAS. In PhotoEngine Studio, open **CAD Standards -> Tier 2 Reference Vault Folder**, browse to that network folder, and click **Save Standards**.

### 2. Initialize Configuration & Extract AutoCAD Bridge

Open Command Prompt, PowerShell, or Git Bash in `C:\Tools\PhotoEngine\` and run:

~~~powershell
# Clear browser Mark-of-the-Web (prevents Windows SmartScreen prompt on downloaded .exe/.zip)
Unblock-File .\PhotoEngine.exe

# Initialize config.json and extract PhotoBridge.lsp
.\PhotoEngine.exe --init
~~~

This creates two items under `%LOCALAPPDATA%\PhotoEngine\`:
1. `config.json` — Local CAD layer standards, text styles, block rotation/multi-head profiles, and library paths.
2. `PhotoBridge.lsp` — AutoLISP bridge connecting AutoCAD to `PhotoEngine.exe`.

### Verifying Release Authenticity (Ed25519 Signature)

Every official release includes `SHA256SUMS.txt`, `SHA256SUMS.txt.sig`, and `KEYS.allowed_signers`. Run these built-in PowerShell commands in your download folder to verify the release came from the official signing key (`SHA256:FbW9u1qenTBLj2G/KXZYQDii8cHw6K5zQznup5MoGfs`):

~~~powershell
cmd /c "ssh-keygen -Y verify -f KEYS.allowed_signers -I elliott.breinholt@photoengine -n file -s SHA256SUMS.txt.sig < SHA256SUMS.txt"
(Get-FileHash .\PhotoEngine.exe -Algorithm SHA256).Hash.ToLower()
Select-String "PhotoEngine.exe" .\SHA256SUMS.txt
~~~

### 3. Link AutoCAD (`APPLOAD` Startup Suite)

1. Open AutoCAD and run the **`APPLOAD`** command.
2. Under **Startup Suite**, click **Contents... -> Add...**
3. Select `%LOCALAPPDATA%\PhotoEngine\PhotoBridge.lsp` (e.g., `C:\Users\<YourUser>\AppData\Local\PhotoEngine\PhotoBridge.lsp`).
4. Click **Close** and load it once into your active drawing (or restart AutoCAD).

#### Primary AutoCAD Commands
- **`PE_UI`** — Exports the current DWG snapshot and opens (or focuses) **PhotoEngine Studio** linked live to AutoCAD.
- **`PE_CALC`** — Runs a direct headless solve and redraws calc points, contours, and templates in the active DWG.

You can also launch `PhotoEngine.exe` directly from Windows Explorer or a taskbar shortcut at any time—it automatically detects an open AutoCAD session via Windows COM or runs in Standalone Mode for quick test rooms.

## Managing IES Libraries & CLI Reference

~~~powershell
# Initialize config.json and extract PhotoBridge.lsp
PhotoEngine.exe --init

# Repack local IES_Library folder + Promoted fixtures into portable IES_Favorites.zip
PhotoEngine.exe --pack-favorites

# Build or refresh .pe_vault_index.tsv.gz across all brand .zips in Vault folder
PhotoEngine.exe --index-vault C:\Tools\PhotoEngine\IES_Vault
PhotoEngine.exe --index-vault auto

# Import a new manufacturer .zip into local IES_Library (Favorites only or full archive)
PhotoEngine.exe --import-zip BrandPhotometry.zip --favorites-only

# Inspect parsed lumens, watts, CCT, CRI, category, and BUG rating of a single .ies file
PhotoEngine.exe --inspect sample.ies

# Solve a project JSON headlessly from the command line
PhotoEngine.exe --calc project_in.json --out project_out.json --draw-lsp draw.lsp
~~~

### Promoting Fixtures from the Vault to Favorites
1. Open **Browse IES Library** in Studio and toggle **`Vault`** on the filter strip.
2. Click the star icon on any `VAULT` row to extract a copy into `IES_Library\<Brand>\Promoted\` and mark it as a Tier 1 Favorite.
3. Click **`Pack Favorites.zip`** in the Library Browser header anytime you want to bake newly promoted fixtures into your portable `IES_Favorites.zip` sidecar.

## Building & Releasing from Source

### Prerequisites
- **Windows 10 / 11 (x64)** with Microsoft Edge WebView2 Runtime
- **Go 1.23+**
- **Python 3.10+** (standard library only; generates multi-size `appicon.ico` and native COFF `.syso` resources)

### Local Build & Portable Packaging
~~~bash
# Build PhotoEngine.exe stamped with active Git tag & commit hash
./build.sh

# Build + bundle PhotoEngine.exe, IES_Favorites.zip, and README.md into dist/
./package_dist.sh
~~~

### Publishing a New GitHub Release
~~~bash
git tag v1.1.0
git push origin master --tags
~~~
GitHub Actions (`.github/workflows/release.yml`) automatically runs unit tests on `windows-latest`, compiles `PhotoEngine.exe`, generates commit release notes, and publishes the GitHub Release. Upload `IES_Favorites.zip` (and optionally a zipped `IES_Vault.zip`) as release assets whenever library contents change.

## License & Engineering Disclaimer

**Copyright © 2026 Elliott W. Breinholt. All rights reserved.**

Proprietary software — unauthorized reproduction, reverse engineering, redistribution, or sublicensing is prohibited without written permission. Photometric outputs are mathematical simulations dependent on manufacturer IES LM-63 files and user-supplied assumptions; the Designer or Professional Engineer of Record remains solely responsible for independent verification and code compliance.

---

## Adding Your Own AutoCAD Lighting Blocks (Settings Tab)

Every engineering firm draws AutoCAD lighting symbols differently. The **Block Mapping** table inside the **Settings** tab connects your CAD block names (`INSERT` entities) to PhotoEngine's photometric solver and throw templates (`E-LITE-TMPL`).

| Setting Field | What It Controls | How to Fill It In |
| :--- | :--- | :--- |
| **Block Name** | Exact AutoCAD block name | Open AutoCAD **Properties (`CH`)** on your fixture symbol and copy its exact `Name` (e.g., `E-LTG-2X4`, `SITE-POLE-2H`). |
| **Mount / Default Z** | Fallback fixture height (ft) | Used when 2D CAD plans place blocks at `Z = 0` (e.g., `9.0` for ceiling troffers, `25.0` for site poles). |
| **Aiming / Rotation Offset (`°`)** | Aligns IES forward throw (`0°`) with your CAD symbol | Insert your block in AutoCAD at **`Rotation = 0`** and see which way the symbol points:<br>• Points **Right (`+X` / East)** → `0°`<br>• Points **Up (`+Y` / North)** → `90°`<br>• Points **Left (`-X` / West)** → `180°`<br>• Points **Down (`-Y` / South)** → `270°` (or `-90°`) |
| **Multi-Head Angles & Arm Offset** | Poles with 2, 3, or 4 luminaire heads | Add one head entry per fixture on the pole:<br>• **Single Head:** `[0°]`<br>• **Twin Back-to-Back:** `[0°, 180°]`<br>• **Triple @ 120°:** `[0°, 120°, 240°]`<br>• **Quad Cross:** `[0°, 90°, 180°, 270°]`<br>*Arm Offset (ft)* shifts each head outward from the pole center (typically `1.0`–`2.0` ft). |

> **Quick Visual Verification:** Enable the **`E-LITE-TMPL`** aiming template toggle and run a quick pass in AutoCAD. If the cyan photometric throw template points 90° or 180° away from your CAD block graphic, adjust **Aiming Offset** in `90°` increments until the template cone projects cleanly out of the front of your symbol.
