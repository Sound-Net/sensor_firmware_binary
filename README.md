# SoundNet sensor firmware binaries

Compiled firmware for the SoundNet sensor package, published so that the
[SoundNet Firmware Updater](https://github.com/Sound-Net/sensorfirmwareupdater)
can offer it to researchers in the field.

**Nobody needs to clone this repository to use it.** The updater application
reads `manifest.json` over HTTPS at startup and downloads whatever firmware the
user selects. Publishing a new firmware release is a commit and a push — every
copy of the updater already installed picks it up the next time it runs.

Firmware source lives in
[`sensor_firmware`](https://github.com/Sound-Net/sensor_firmware). This
repository holds only the build products.

## Layout

```
manifest.json                 <- the index the updater reads
firmware/                     <- the flashable images it refers to
├── soundnet_firmware-2.0.7-SOUNDNET_V1_R5.hex
└── soundnet_firmware-2.0.7-SOUNDNET_V1_R6.hex

SparkFun.avr.promicro_V1_R5/  <- raw Arduino build output, kept for reference
SparkFun.avr.promicro_v2_R6/
```

Two rules, and they are the whole contract with the updater:

1. `manifest.json` sits at the repository root.
2. Every `file` it names sits in `firmware/`, as a bare file name.

The `SparkFun.avr.*` directories are the untouched Arduino build output. They
are not read by the updater; they are kept because the `.elf` files are useful
when debugging a field failure, and `.with_bootloader.hex` is what you need when
programming a bare board over ISP rather than over USB. Only the plain
`soundnet_firmware.ino.hex` — the application without the bootloader — is
suitable for the updater.

**Old firmware is never deleted.** Researchers sometimes need to put a sensor
back onto the version the rest of a deployment is running, and a `.hex` is about
70 KB.

## `manifest.json`

```json
{
  "schemaVersion": 1,
  "updated": "2026-08-31",
  "firmware": [
    {
      "revision": "SOUNDNET_V1_R6",
      "version": "2.0.7",
      "file": "soundnet_firmware-2.0.7-SOUNDNET_V1_R6.hex",
      "sha256": "440a5394…",
      "released": "2026-08-31",
      "notes": "Fixes the random field lockups that needed a power cycle to clear.",
      "recommended": true
    }
  ]
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `revision` | yes | A `SENSOR_TYPE` name from `xsensmessage.h`. Unknown names are skipped with a warning. |
| `version` | yes | The `FIRMWARE_VERSION` string the sensor reports over serial. |
| `file` | yes | Bare file name inside `firmware/`. Any directory part is stripped. |
| `sha256` | strongly recommended | Checked after download; a mismatch is refused rather than flashed. |
| `released` | no | Shown under the drop-down in the updater. |
| `notes` | no | One line, shown under the drop-down. Write it for a researcher, not a developer. |
| `recommended` | no | Sorts to the top of the list. Mark the newest build of each revision. |

`revision` and `version` together identify a build; publishing the same pair
twice replaces the earlier entry.

### Why `sha256` matters

These bytes are written into a sensor's flash, often somewhere that nobody can
easily re-flash. A truncated download would produce a device that does not run.
The updater refuses anything whose hash does not match and says so plainly. An
entry with no `sha256` still installs, but the updater logs a warning.

## Publishing a new release

From a checkout of the firmware source repository, alongside this one:

```bash
tools/build-firmware.sh --out ../sensor_firmware_binary SOUNDNET_V1_R5 SOUNDNET_V1_R6
```

That compiles once per hardware revision, writes the `.hex` files into
`firmware/`, computes the hashes, and merges them into `manifest.json` — keeping
older entries whose files are still present and marking the newest build of each
revision as recommended. Then:

```bash
git add -A && git commit -m "Firmware 2.0.7" && git push
```

## A note on hardware revisions

`SENSOR_TYPE` is a compile-time constant, so each hardware revision needs its
own build. For most revisions it changes real pin assignments (`XSENS_RX`,
`XSENS_TX`, `XSENS_PWR`) and installing the wrong one leaves a sensor that
powers up but never reads its Xsens.

**R5 and R6 are the exception.** `globals.h` maps both to the same pins, so the
two 2.0.7 images differ by a single byte: the device type the sensor reports
over serial. Mixing those two up is untidy but harmless. Do not assume that
holds for any other pair.
