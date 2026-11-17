# Building WaterPyDesktop.exe and setup.exe

The Windows executable must be built on Windows. Two ways:

## A. On a Windows PC
1. Install 64-bit **Python 3.12** and (optional, for setup.exe) **NSIS 3** from https://nsis.sourceforge.io.
2. Double-click `build_windows.bat`.
3. Results:
   * `dist\WaterPyDesktop\WaterPyDesktop.exe` — the application folder (copy the whole folder)
   * `dist\WaterPyDesktop-<version>-setup.exe` — the installer (only when NSIS is installed)

## B. Online with GitHub Actions (no Windows PC needed)
The repository already contains `.github/workflows/build-windows.yml`.
1. Upload the project to a GitHub repository (with the hidden `.github` folder).
2. **Actions → Build Windows EXE and setup.exe → Run workflow**.
3. After ~8 minutes download the artifact: `WaterPyDesktop-<version>-setup.exe` and
   `WaterPyDesktop-<version>-portable.zip`.
4. Pushing a tag such as `v0.6.0` also creates a GitHub Release with both files attached.

### From an iPhone (upload the zip instead of the folder)
Safari cannot upload folders. Upload the source **zip** with *Add file → Upload files*, then create the file
`.github/workflows/build-from-zip.yml` with *Add file → Create new file* and paste:

```yaml
name: Build from uploaded zip
on: workflow_dispatch
jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Unzip source
        shell: pwsh
        run: |
          $zip = Get-ChildItem -Filter "WaterPyDesktop-source-*.zip" | Sort-Object Name | Select-Object -Last 1
          Expand-Archive -Path $zip.FullName -DestinationPath src
          $dir = Get-ChildItem src -Directory | Select-Object -First 1
          "APPDIR=$($dir.FullName)" | Out-File -FilePath $env:GITHUB_ENV -Append
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Build EXE and setup.exe
        shell: pwsh
        working-directory: ${{ env.APPDIR }}
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r requirements.txt
          python -m PyInstaller --noconfirm --clean --windowed --onedir --name WaterPyDesktop `
            --icon waterpy\resources\waterpy.ico --add-data "waterpy\resources;waterpy\resources" `
            --add-data "plugins;plugins" --add-data "sdk;sdk" --add-data "examples;examples" `
            --add-data "README.md;." --add-data "PLUGIN_SDK.md;." --add-data "CHANGELOG.md;." `
            --collect-submodules waterpy main.py
          if (-not (Get-Command makensis -ErrorAction SilentlyContinue)) { choco install nsis -y --no-progress }
          $makensis = (Get-Command makensis -ErrorAction SilentlyContinue).Source
          if (-not $makensis) { $makensis = "${env:ProgramFiles(x86)}\NSIS\makensis.exe" }
          $v = (Get-Content VERSION).Trim()
          Push-Location installer; & $makensis /V2 "/DVERSION=$v" WaterPyDesktop.nsi; Pop-Location
          Compress-Archive -Path dist\WaterPyDesktop\* -DestinationPath "dist\WaterPyDesktop-$v-portable.zip"
      - uses: actions/upload-artifact@v4
        with:
          name: WaterPyDesktop-windows
          path: |
            ${{ env.APPDIR }}/dist/*-setup.exe
            ${{ env.APPDIR }}/dist/*-portable.zip
```
Then **Actions → Build from uploaded zip → Run workflow**. For a new version, upload the new zip
(the newest `WaterPyDesktop-source-*.zip` is used automatically).

## Notes
* The installer asks for administrator rights, installs to `C:\Program Files\WaterPyDesktop`, adds Start-menu
  and (optional) desktop shortcuts, registers `.wpplugin` / `.ui` file types, and appears in
  *Settings → Apps* with an uninstaller.
* The installed IDE still needs **Python 3.12 + `pip install PySide6 pyinstaller`** to *run* and *build* the
  apps you design (the IDE finds `python`, `py -3.12`, or a project `.venv`).
* The EXE is not code-signed, so Windows SmartScreen may warn on first launch ("More info → Run anyway").
  Sign `WaterPyDesktop.exe` and the setup with `signtool` if you have a certificate.

## Installers for apps you create
Inside the IDE: **Run → Setup.exe** (Ctrl+Shift+B). It builds your app with the settings from
**Project → Build Settings** (name, version, icon, one-file/folder, console, hidden imports) and packages it
with **Inno Setup 6** or **NSIS 3** (whichever is installed) into `build_output\installer\<App>-<version>-setup.exe`.
If neither compiler is installed, the `.iss` and `.nsi` scripts are still written so you can compile them later.
