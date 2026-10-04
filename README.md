# SaveSync3DS

A muOS application that moves Nintendo DS save files between a handheld running
DraStic and a Nintendo 3DS running [ftpd](https://github.com/mtheall/ftpd),
over your local network. No PC, no card swapping, no cables.

Pick a game, pick a direction, done. The save is backed up before anything is
overwritten and read back afterwards to verify it landed.

Written in LÖVE 11.5 (UI) and Python 3 standard library only (transfers), so it
runs on a stock muOS install with the PortMaster runtime.

## Why this is not just an FTP copy

The two sides store DS saves differently, and copying the bytes across gets you
a corrupt save:

- **The 3DS** runs DS games through TWiLight Menu++ / nds-bootstrap, which keeps
  the save in a `saves` folder next to the ROM, named after the ROM, and **pads
  it to a fixed size** (commonly 512 KB where DraStic writes 64 KB).
- **TWiLight's per-game "save number"** chooses which file is live: number 0 is
  `<rom>.sav`, number N is `<rom>.savN`. DraStic has no such concept — one save
  per game.
- **DraStic** writes either a raw `<rom>.sav` (drastic-legacy) or a `<rom>.dsv`
  (drastic-trngaje with `backup_use_sav_format = 0`), which is the raw data plus
  a DeSmuME footer.

SaveSync3DS handles all of that:

- Push reads whichever of `.sav` / `.dsv` is newer and pads or trims it to the
  size nds-bootstrap expects for that game's other slots.
- Pull always writes a raw `.sav` and moves any `.dsv` out of the way, so
  drastic-trngaje imports the new save rather than its own older copy.
- The slot you are writing to is shown along with the slot TWiLight is currently
  set to, so you do not quietly write to one the 3DS is not reading.
- Every transfer is read back and compared. If it does not match, the backup is
  kept and the operation fails loudly.

## Requirements

**On the handheld**

- muOS (tested on an RG35XX-H, aarch64)
- PortMaster, for its `love_11.5` runtime and `gptokeyb2`
- Python 3 — `/usr/bin/python3` on stock muOS
- DraStic, with saves in `/mnt/mmc/MUOS/save/drastic/backup`

**On the 3DS**

- ftpd running, with your DS ROMs visible under `/roms/nds`
- TWiLight Menu++ / nds-bootstrap with `SAVE_LOCATION = 0` (saves next to ROM)

Both devices on the same network.

## Install

Copy the contents of [`app/`](app) to:

```
/mnt/mmc/MUOS/application/SaveSync3DS/
```

Copy the icons so muOS draws it properly:

```
icons/glyph/savesync3ds.png  ->  <theme>/glyph/muxapp/savesync3ds.png
icons/grid/savesync3ds.png   ->  <theme>/image/grid/muxapp/savesync3ds.png
```

Then launch **SaveSync3DS** from Applications.

> If you install somewhere other than `MUOS/application/SaveSync3DS`, change
> `APP_DIR` in `mux_launch.sh` to match.

## First run

Start ftpd on the 3DS — it prints its IP address on screen. The app opens the
IP editor on first run; enter that address and press A. It is saved to
`config.ini`, so you only do this once.

The 3DS IP will change if your router hands out a new DHCP lease. Press **X** any
time to edit it again, or give the 3DS a DHCP reservation.

## Controls

| Button    | Game list                     | Transfer menu              | IP editor              |
| --------- | ----------------------------- | -------------------------- | ---------------------- |
| D-pad ↑↓  | Move one game                 | Move                       | Change digit by 1      |
| D-pad ←→  | Move one page                 | Change 3DS save slot       | Move between octets    |
| A         | Open the selected game        | Confirm                    | Accept the address     |
| B         | Quit                          | Back to the list           | Cancel                 |
| X         | Edit the 3DS IP               | —                          | —                      |
| Y         | Test the connection           | —                          | —                      |
| L1 / R1   | —                             | —                          | Change digit by 10     |

## Configuration

Everything lives in **[`app/config.ini`](app/config.ini)**, a flat `key=value`
file read by both halves of the app. Every key is optional and the shipped file
documents each one — a stock muOS and stock ftpd setup only needs `ip`.

| Key                | Default                              | What it is                                       |
| ------------------ | ------------------------------------ | ------------------------------------------------ |
| `ip`               | *(asked on first run)*               | Your 3DS, as shown by ftpd                       |
| `port`             | `5000`                               | ftpd's port                                      |
| `user`             | `anonymous`                          | ftpd login, if you set one                       |
| `pass`             | *(empty)*                            | ftpd password, if you set one                    |
| `timeout`          | `10`                                 | Seconds to wait for the 3DS                      |
| `remote_rom_root`  | `/roms/nds`                          | Where ftpd exposes your DS ROMs                  |
| `save_dir`         | `/mnt/mmc/MUOS/save/drastic/backup`  | DraStic's save folder on the handheld            |
| `backups_kept`     | `10`                                 | Timestamped backups kept before the oldest drops |
| `rom_dirs`         | the four usual muOS paths            | Where to find DS ROMs locally, comma separated   |
| `python`           | `/usr/bin/python3`                   | Interpreter used to run `sync3ds.py`             |

Any key can also be set as an environment variable, which wins over the file:

```sh
SYNC3DS_PORT=5001 SYNC3DS_REMOTE_ROM_ROOT=/nds python3 sync3ds.py ping 10.0.0.5
```

Changing the IP in the app rewrites only the `ip=` line — your comments and
other settings are preserved.

### Running the backend on its own

`sync3ds.py` is a standalone script with no dependencies and is useful for
debugging from SSH:

```sh
python3 sync3ds.py ping   <ip>
python3 sync3ds.py status <ip> <rom base name>
python3 sync3ds.py push   <ip> <rom base name> [slot]   # handheld -> 3DS
python3 sync3ds.py pull   <ip> <rom base name> [slot]   # 3DS -> handheld
```

It prints `key=value` lines, ending with `ok=1|0` and a one-line `msg=`.

## Backups

Before anything is overwritten, the existing save is copied to:

```
<save_dir>/sync3ds_backups/
```

with a timestamp and a marker for which side it came from. The newest
`backups_kept` are retained. If a transfer fails to verify, the backup is left
in place and the app says so.

## Notes and gotchas

- ftpd reports a missing file as **`450 No such file or directory`**, where
  almost every other FTP server uses 550. Both are treated as "not found".
- Both devices idle-sleep and drop the connection. Press **Y** to re-test before
  assuming something is broken.
- `reference/drastic-trngaje_drastic.cfg` is a known-good DraStic config for the
  `.dsv` save path, included because the relevant key
  (`backup_use_sav_format = 0`) is easy to get wrong.

## Licence

[MPL-2.0](LICENSE)
