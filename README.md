# ADI IIO contributions

This repo is where I keep track of my open source work on the Analog Devices
IIO stack. The code itself lives upstream. This is the log of what I picked
up, why, what happened to it and what I learned along the way.

## Background

IIO (Industrial I/O) is the Linux kernel subsystem that most ADI converters,
sensors and RF transceivers sit behind. On top of it there are two projects I'm
working in:

- [libiio](https://github.com/analogdevicesinc/libiio) is the C library (with
  Python bindings) that talks to IIO devices, locally or over the network.
- [pyadi-iio](https://github.com/analogdevicesinc/pyadi-iio) is the Python layer
  with one class per part (`adi.ad9361`, `adi.adxl355` and so on), which is
  what most people actually import.

I studied ECE and most of my projects are on the embedded and signal chain
side, so these are libraries I'd end up using anyway. Contributing to them
seemed like a better way to learn them than just reading the docs.

## Contributions

| Repo | Issue / PR | What | Status |
|------|------------|------|--------|
| pyadi-iio | [#808](https://github.com/analogdevicesinc/pyadi-iio/pull/808) | `ada4961` and `adf4382` raised errors naming the wrong part (ADF4159 / ADF4355) | Open, waiting for review |
| pyadi-iio | [#96](https://github.com/analogdevicesinc/pyadi-iio/issues/96) | Older classes accept a `None` device from `find_device()` and fail later with confusing errors | [Asked](https://github.com/analogdevicesinc/pyadi-iio/issues/96#issuecomment-6544582633) which approach they want, waiting for a reply |
| pyadi-iio | [#809](https://github.com/analogdevicesinc/pyadi-iio/pull/809) (fixes [#489](https://github.com/analogdevicesinc/pyadi-iio/issues/489)) | Write only test helpers rejected write only properties, because `hasattr()` calls the getter | Open, waiting for review |
| pyadi-iio | [#638](https://github.com/analogdevicesinc/pyadi-iio/issues/638) | Stray `print` in the AD7746 class, already fixed on `main` by `b5ec490` | [Commented](https://github.com/analogdevicesinc/pyadi-iio/issues/638#issuecomment-6544463989), waiting for close |
| pyadi-iio | [#585](https://github.com/analogdevicesinc/pyadi-iio/issues/585) | AD5686 read back mV but took V, already fixed on `main` by `7d9be84` (and `volts` became `voltage`) | [Commented](https://github.com/analogdevicesinc/pyadi-iio/issues/585#issuecomment-6544480249), waiting for close |
| libiio | [#1613](https://github.com/analogdevicesinc/libiio/issues/1613) | Python bindings crash on import on Windows, `find_library("c")` returns `None` | **Fixed** by the maintainer in [#1619](https://github.com/analogdevicesinc/libiio/pull/1619) (merged 2026-11-05), using my option 1 with option 2 as fallback. Not in a release yet |

## Setup

I don't have any ADI boards, so everything here is tested with mocked
contexts or the iio-emu emulator rather than real hardware. The PR template
asks you to say which one you used, so I always note it.

On Windows 11:

- Fork and clone pyadi-iio, make a venv, `pip install -r requirements_dev.txt`
  and `pip install -e .`
- pylibiio needs the libiio C library. I took the MSVC build from the libiio
  v1.0.0 release (`Windows.zip`) and copied the DLLs into `.venv/Scripts`
- That still failed on import, see the libiio bug below. I patched my local
  copy of pylibiio to get past it
- The pre-commit hooks pin black 19.10b0, which needs `typed-ast`, which won't
  build on Python 3.13. CI lints on 3.10, so I use a second venv on 3.10
  (installed with `uv python install 3.10`) just for pre-commit

The pure unit tests run fine on Windows like this. The emulated tests
(`pytest --emu`) need iio-emu, which CI runs on Linux, so those will go
through WSL.

## Progress log

### 2026-10-30

- Forked pyadi-iio and got a dev environment working on Windows.
- Baseline on a clean `main`: all pre-commit hooks pass, and the mocked unit
  tests (`test_refactored_devices_unit.py`, `test_trigger.py`,
  `test_ad6676.py`) give 55 passed, 30 skipped. The skips need emu or
  hardware.
- Getting `import iio` to work on Windows turned up a real bug in the libiio
  Python bindings. `iio/__init__.py` loads libc with
  `CDLL(find_library("c"))`, but on Windows `find_library("c")` returns
  `None`, so the import dies with a `TypeError`. The only thing it needs libc
  for is `free()` on the string from `iio_context_get_xml()`. The catch is
  that the MSVC build of `libiio1.dll` links the UCRT while the MinGW build
  links `msvcrt.dll`, so there's no single right answer to hardcode. I've
  drafted an issue rather than a PR so the maintainers can pick the approach.
- Went through the open pyadi-iio issues that don't need hardware. #638 and
  #585 are already fixed on `main` (by `b5ec490` and `7d9be84`).
- **Plan change:** I was going to start with #96 (raise a clear error when
  `find_device()` returns `None`). Looking closer, the refactored classes in
  `rx_tx.py` already do this, and it's only the older classes that don't. The
  maintainers are moving those onto `device_base` one by one, so a 144-line
  sweep might clash with that. I'm asking on the issue first. While
  reproducing it, I found that `ada4961` and `adf4382` raise errors naming
  the wrong part, which is small and uncontroversial, so that became the
  first PR instead.
- Fix branch pushed to my fork with a mocked test. The test fails before the
  fix and passes after it.

### 2026-10-31

- Rechecked everything against upstream before posting. Nothing had moved,
  and the fix branch still merges cleanly.
- Posted my first comments upstream, on #638 and #585, pointing at the
  commits that already fixed them so they can be closed. I kept the #585 one
  short but still mentioned the `volts` to `voltage` rename, since that will
  break old scripts.
- Opened my first PR, #808, for the wrong part names. I kept the description
  short, filled in the repo's template, and said clearly that it was tested
  with a mocked context, not hardware.
- Commented on #96 before writing any code. I re-ran my reproduction on the
  latest `main` first, and it still holds: `ad9094` and `ad9136` construct
  with no device at all, and `ad9625` and `ad9250` fail with an unrelated
  `TypeError`. I offered two options: a small shared helper rolled out a few
  classes at a time, or leaving it to the `device_base` migration. Since the
  issue has been assigned to someone else since 2020, I also asked whether
  anyone is still working on it.
- Filed libiio #1613 for the Windows import crash. Before posting I checked
  it on Python 3.10 as well as 3.13 (same `TypeError` on both), confirmed the
  line is still on libiio `main`, and searched for duplicates. The closest
  ones were the same lookup problem on embedded Linux and macOS, both fixed
  separately. The issue lays out three fixes: export a free function from
  libiio, hardcode `ucrtbase` for the official MSVC builds, or skip the free
  on Windows as a stopgap. I left the choice to the maintainers, since
  exporting a free function would mean adding to the C API.
- Second PR, #809, for #489. The issue was only a link to a line, so I had
  to work out the bug myself. The write only test helpers check
  `hasattr(sdr, attr)` first, but `hasattr()` works by reading the
  attribute, and a write only property can't be read. So the helpers
  rejected exactly the properties they were written for. I scanned the
  library and found 15 such properties (`ad9081.pfilt_config` and others)
  that couldn't be tested this way. The fix checks the class instead. My
  unit test with a fake write only device fails 3 of 4 cases before the fix
  and passes all 4 after. I also checked that the two existing callers still
  find their attributes.
- Possible next item: ad7746's write only test seems to replace a method on
  the channel object instead of calling it, so nothing gets written to the
  device. Need to confirm before raising it.

### 2026-11-04

- Got a reply on libiio #1613 from one of the maintainers. He confirmed the
  Windows crash is a real bug and is going to fix it himself. He went with
  option 1 (a free function exported from libiio) and wants option 2
  (`ucrtbase`) as the fallback. So no PR from me on this one, but the
  diagnosis held up and the fix follows what I proposed.
- Thanked him and offered to test his branch on Windows, since I already
  have both the MSVC and MinGW builds set up from when I tracked this down.

### 2026-11-05

- libiio #1613 is closed. The maintainer's fix (#1619) got merged this
  morning. It adds `iio_free()` to libiio so the bindings free memory with
  libiio's own C runtime, and falls back to `ucrtbase` on Windows for older
  libiio builds, with `LIBIIO_PY_CRT=msvcrt` for MinGW users. That's the
  option 1 plus option 2 combination from my issue. It also turned out an
  earlier PR (#1615) had patched the import with `msvcrt.dll`, which suits
  MinGW but not the official MSVC builds, so the issue caught that too.
- It merged before I got to test it, so no test report from me. It'll ship
  in the next libiio release after v1.0.0.
