# RESOGUN notes

Findings from the first look at the game files. Target build: **CUSA00038 01.10**.

## The dump

- Content ID `UP9000-CUSA00038_00-RESOGUN000000002`.
- Base game 01.00 (446 MB PKG) plus the 01.10 update (642 MB PKG). Both are fake PKGs: the
  EKPFS decrypts with the public fake keys, so `scripts/pkg_extract.py` unpacks them.
- 01.10 changelog (Sony): PS4 Pro support, 4K on Pro, HDR, visual improvements.

Files in the 01.10 update:

| File | Size | What it is |
|---|---|---|
| `eboot.bin` | 17.4 MB | fSELF, plaintext segments. ELF type `0xFE10` (SCE dynamic executable), entry `0x36cbe0` |
| `sce_module/libc.prx` | 1.06 MB | The game's own libc (linked by `link_libc.py`) |
| `sce_module/libSceFios2.prx` | 451 KB | The game's own Fios2 (linked by `link_modules.py`) |
| `audio.pck` | 287 MB | Wwise file package (`AKPK`) |
| `stream.dat` | 353 MB | Housemarque data archive (`TSFM` magic), not documented yet |
| `sce_discmap.plt`, `sce_discmap_patch.plt` | | Disc maps |
| `video/title.mp4`, `video/menu.mp4` | | Intro and menu videos (base game only) |

The update replaces `eboot.bin`, both PRX files, `audio.pck` and `stream.dat`; the videos come
from the base game, so the game folder is the base game with the update copied over it.

## The executable

- Engine: Housemarque **Sasquatch**. The build path in the asserts is
  `D:\jenkins\workspace\RESOGUN eX PS4\resogun_EX\...`; the render backend lives in
  `Libs\Sasquatch\Impl\Render\Lowlevel\Orbis\`.
- Segments: code `0x0`-`0xc42604` (R E), relro `0xc44000`, data `0xd08000` (filesz `0x1c4ec8`,
  memsz `0x89f0e0`), TLS 0x10 bytes, `.eh_frame_hdr` at `0xbf3710`.
- 74,512 RELA relocations, 734 PLT relocations, 751 imported symbols.
- Audio is Wwise on `libSceAudio3d` + `libSceAudioOut`, the same pairing as WipEout.

### Imported libraries (50)

libkernel, libc, libScePosix, libSceFios2, libSceSysmodule, libSceSystemService,
libSceUserService, libSceGnmDriver, libSceVideoOut, libSceAvPlayer, libSceAjm, libSceAudioOut,
libSceAudio3d, libScePad, libSceMove, libSceMoveTracker, libSceCamera, libSceHmd,
libSceHmdSetupDialog, libSceVrTracker, libSceSaveData, libSceSaveDataDialog, libSceAppContent,
libSceRtc, libSceFiber, libSceJson2, libSceScreenShot, libSceIme, libSceImeDialog,
libSceCommonDialog, libSceErrorDialog, libSceMsgDialog, libSceWebBrowserDialog,
libSceInvitationDialog, libSceNpFriendListDialog, libSceCompanionHttpd, libSceHttp, libSceSsl,
libSceNet, libSceNetCtl, libSceNpAuth, libSceNpCommerce, libSceNpCommon, libSceNpManager,
libSceNpMatching2, libSceNpScore, libSceNpTrophy, libSceNpTus, libSceNpUtility, libSceNpWebApi.

## Import check (first build)

Built on Linux (Ubuntu 24.04, `GOW3_LTO=OFF bash build.sh`) and run with
`run.py --check-imports` on 01.00 + 01.10. Preparation works unchanged: the game's own libc and
Fios2 link at `0x15b0000` / `0x16b0000` (203 native bindings, `fs`->`gs` patched 10 times),
leaving 644 imports to the host runtime. **105 of them are missing.**

AvPlayer (the intro videos) is covered by shadPS4's library in the video core; Move and Camera
answer "not connected" (`src/runtime_extra.c`). Gnm, VideoOut, AudioOut, Audio3d, Pad, SaveData
and the rest of libkernel are covered.

Missing, by library (names from `tools/aerolib.inl`):

**Needed for the game to run**

- libSceFiber (6): `_sceFiberInitializeImpl`, `sceFiberFinalize`, `sceFiberGetSelf`,
  `sceFiberReturnToThread`, `sceFiberRun`, `sceFiberSwitch`. No implementation in the runtime
  yet; needs real context switching.
- libSceJson2 (13): `sce::Json::Value::set` (ValueType, String, bool, double, long, unsigned
  long), `Value::count`, `Value::toString`, `Value::operator bool`, `Value::operator[](unsigned
  long)`, `String::at`, `String::length`, `String::operator==(const char*)`. Extends
  `src/runtime_json.c`.
- libkernel (3): `sceKernelSleep`, `scePthreadGetthreadid`, `scePthreadSetschedparam`.
- libSceRtc (3): `sceRtcParseDateTime`, `sceRtcTickAddHours`, `sceRtcTickAddSeconds`.
- libSceAppContent (2): `sceAppContentGetAddcontInfo`, `sceAppContentTemporaryDataMount2`.
- libScePad (1): `scePadSetMotionSensorState`.
- libSceUserService (1): `sceUserServiceGetUserColor`.
- libSceSystemService (1): `sceSystemServiceReenableMusicPlayer`.

**Stub as unavailable (PS VR, PSN, companion app, dialogs)**

- libSceVrTracker (11) and libSceHmd (4): PS VR. Report no headset.
- libSceCompanionHttpd (10): the second-screen companion app server.
- libSceNpScore (8) and libSceNpTus (10): leaderboards and title user storage.
- libSceNpManager (4), libSceNpUtility (3, word filter), libSceNpMatching2 (1),
  libSceNpCommerce (1), libSceNpFriendListDialog (6), libSceInvitationDialog (1),
  libSceWebBrowserDialog (5), libSceErrorDialog (1).
- libSceIme (3), libSceImeDialog (1): on-screen keyboard.
- libSceHttp (2), libSceNet (1), libSceScreenShot (3).

The check lists imports only; the game has not been started yet.

## shadPS4 reports for this game

- Linux, shadPS4 0.12.5, 01.10: playable, no graphics or speed issues, ship editor works
  ([#1934](https://github.com/shadps4-compatibility/shadps4-game-compatibility/issues/1934)).
- Windows, up to shadPS4 0.21.0: crashes after the splash screen with the SDL audio backend;
  reported working with OpenAL. The log shows a dummy `sceSysmoduleLoadModule` for
  `ORBIS_SYSMODULE_AV_PLAYER`
  ([#413](https://github.com/shadps4-compatibility/shadps4-game-compatibility/issues/413)).

So the shadPS4 video core already draws this game; the work here is the runtime side.
