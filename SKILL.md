# DSH Model Reasoning Plugins

Use this skill to install, verify, update, and remove the DSH model reasoning plugin release packages from this repository.

## Package contents

- `model-selection-fix.zip`: fixes model selection behavior.
- `model-slider.zip`: adds model parameter slider controls.
- `reasoning-slider.zip`: adds reasoning parameter slider controls.
- `SHA256-files.json`: checksums for the files inside each plugin archive.

## Prerequisites

- A DSH installation with its plugin directory identified.
- PowerShell 5+ on Windows.
- Permission to write to the DSH plugin directory.
- Do not install two copies of the same plugin.

## Verify the release

Run from the repository root. The manifest checks files inside the nested archives, so extract each archive to a temporary directory before checking hashes.

```powershell
$repo = (Get-Location).Path
$tmp = Join-Path $env:TEMP ("dsh-plugin-verify-" + [guid]::NewGuid())
New-Item -ItemType Directory -Path $tmp | Out-Null

Get-ChildItem $repo -Filter *.zip | ForEach-Object {
  Expand-Archive -LiteralPath $_.FullName -DestinationPath $tmp -Force
}

$manifest = Get-Content (Join-Path $repo 'SHA256-files.json') -Raw | ConvertFrom-Json
$manifest.psobject.Properties | ForEach-Object {
  $file = Join-Path $tmp $_.Name
  if (-not (Test-Path -LiteralPath $file)) { throw "Missing file: $($_.Name)" }
  $actual = (Get-FileHash -LiteralPath $file -Algorithm SHA256).Hash.ToLowerInvariant()
  if ($actual -ne $_.Value.ToLowerInvariant()) { throw "Checksum mismatch: $($_.Name)" }
}

"All release checksums passed."
Remove-Item $tmp -Recurse -Force
```

Do not install a package if verification fails. Re-download the complete release instead.

## Install a plugin

1. Verify the release using the procedure above.
2. Choose the target DSH plugin directory for the current installation.
3. Extract exactly one plugin archive into that directory, preserving its top-level folder.

```powershell
$pluginZip = Join-Path (Get-Location) 'model-slider.zip'
$pluginDir = 'C:\path\to\dsh\plugins'
Expand-Archive -LiteralPath $pluginZip -DestinationPath $pluginDir -Force
```

Replace `model-slider.zip` with `reasoning-slider.zip` or `model-selection-fix.zip` as needed. Restart DSH after installation so the plugin entry point is loaded.

## Update a plugin

Verify the new release first, then extract the replacement archive with `-Force` over the existing plugin directory. Keep a backup of the old plugin folder until DSH starts successfully. If the update fails, restore the backup and restart DSH.

## Remove a plugin

Remove only the plugin folder that matches the archive name, for example:

```powershell
Remove-Item 'C:\path\to\dsh\plugins\model-slider' -Recurse -Force
```

Confirm the path before running `Remove-Item`. Restart DSH after removal.

## Troubleshooting

- **Checksum mismatch:** discard the local release and obtain a fresh copy from the repository.
- **Plugin is not loaded:** confirm the archive was extracted into the DSH plugin directory with its top-level folder intact, then restart DSH.
- **Duplicate behavior:** remove older copies of the same plugin before restarting.
- **Broken update:** restore the previous plugin folder from backup and restart DSH.

Never edit files inside the ZIP archives while installing. Changes should be made in source and published as a new release package with an updated `SHA256-files.json`.
