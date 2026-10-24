# XCOM 2: War of the Chosen — co-op campaign notes

Running log. Observations, derived facts, guesses and unknowns are kept in separate lists.

## Goal (owner's design)
- Shared strategy layer (Avenger): both players can pick research/build/facility upgrades and
  edit any soldier.
- Squad select: each player picks their own squad (e.g. 4 + 4 = 8 on the field);
  a soldier picked by one player is locked for the other.
- Tactical: each player controls only their own soldiers; players alternate.
- 2 players, Steam, WotC + all DLC.

## Session 1 — 2026-10-24: triage + research

### Observed
- Game identity: see `local/identity.md` (Steam build 8270065, changelist 699444, UE3).
- Team Dragonpunk "XCOM 2: Tactical Co-Op", Workshop 792693587: status "Final", created
  3 Nov 2016, last updated 3 Sep 2017, 61.8 MB. Two-player tactical co-op on campaign
  missions, players split the squad; can import soldiers from the friend's game; no
  Ironman; both players must be in Squad Select when inviting. Team moved to other games.
  Source: https://steamcommunity.com/sharedfiles/filedetails/?id=792693587
- No public source repo found (GitHub repo search "xcom2 coop", "dragonpunk xcom": 0 results).
- robojumper's "WOTC Multiplayer mods" (Workshop 1571537500) soft-deprecated in favour of a
  X2WOTCCommunityHighlander feature for mods in multiplayer.
  Source: https://steamcommunity.com/sharedfiles/filedetails/?id=1571537500
- Steam app 602410 = "XCOM 2 War of the Chosen SDK" (installdir name in steamapps).

### Searched (nothing more found)
- web: "XCOM 2 co-op campaign mod", "Team Dragonpunk ... github source",
  "WOTC shared campaign multiplayer strategy layer XComGameStateNetworkManager"
- GitHub API repo search: "xcom2 coop", "dragonpunk xcom"
- coop-anything DIRECTORY/games/issues: no XCOM entry

### Unknown
- Whether Tactical Co-Op works on WotC (last update 5 days after WotC release; probably vanilla).
- Whether its Workshop package ships `Src/` (many XCOM 2 mods do).
- How the vanilla MP layer (XComGameStateNetworkManager) sends game states — read SDK source.

## Session 1 (cont.) — SDK source reading

SDK: Steam app 602410 "XCOM 2 War of the Chosen SDK", buildid 3075887, 4137 .uc files in
`<SDK>/Development/SrcOrig`. Community Highlander (MIT, github.com/X2CommunityCore/
X2WOTCCommunityHighlander, pushed 2026-10-12) cloned for reference.

### Observed (source, WotC SDK SrcOrig/XComGame/Classes)
- `XComGameStateNetworkManager` (native(MP), script-callable): CreateServer/CreateClient(URL),
  SendHistory, SendPartialHistory, SendGameState, SendMergeGameState, SendGameStateContext,
  SendRemoteCommand(string, array<byte>) + AddCommandParam_Int/String/Bool helpers,
  SendLoadTacticalMap, SendPlayerReady, SendSyncPoint, validation/desync reports.
  Delegates for every receive. `HasServerConnection/HasClientConnection`.
- `X2GameRuleset.SubmitGameStateContext`: if NetworkMgr.HasConnections() and the context
  was user input (`bSendGameState`), it is queued for sending; remote contexts
  (`bNetworkAdded`) reuse the sender's random seed.
- Shipped config `XComGame/Config/DefaultGame.ini:105 bPauseGameStateSending=true`:
  MP uses "active player is authority": active player runs the ruleset; at end of turn
  `SendPartialHistory`, remote replies "HistoryReceived", then "SwitchTurn"; the inactive
  player watches via `XComMPReplayMgr` (X2TacticalMPGameRuleset.uc ~300-360, 1095-1275).
- Remote-ness = `XComGameState_Player.GetGameStatePlayerNetId() != local UniqueId`
  (X2TacticalMPGameRuleset.GameStatePlayerOrGroupIsRemote).
- Transport: `UIMPShell_Lobby.CreateServer(Port=7777, bSteam)`: plain IP, or Steam sockets
  when URL op "steamsockets" is added. Engine NetworkDevice = IpNetDriverSteamworks.
- Cheat exec commands (XComCheatManager.uc 3697-4091): MPCreateServer, MPCreateClient,
  MPSendHistory, MPSetPauseGameStateSending, X2PrintNetworkDebug, MPCreateLobby, ...

### Derived (design direction, not yet tested)
- Tactical: keep ONE XCom player; split the XCom turn into sub-phases A then B. The
  sub-phase owner is the authority; handoff = SendPartialHistory + replay (same as MP
  end-turn). Alien/Lost/Chosen turns run on the host. Units carry an owner tag.
- Strategy: host-authoritative save; client gets full history on join (SendHistory).
  Edits via a "baton": one player edits at a time, handoff sends history. Later: per-object
  locks + SendMergeGameState for parallel soldier editing.
- Test transport: IP on 127.0.0.1, two instances on one PC (unknown if the exe allows it).

### Unknown
- Can two XCom2.exe instances run at once on one PC/Steam account?
- Does SendHistory work while in the strategy layer (Avenger), and does the client's
  visual state (Avenger, Geoscape) refresh after receiving it?
- Where test runs write config/logs (likely %USERPROFILE%\Documents\My Games\...).

## Session 1 (cont.) — running test instances

### Observed
- Owner's install had a corrupt `XComGame/CookedPCConsole/TacticalResources_SF.upk`
  (crash at shell load: "Texture2D AdventDeskA.Textures.AdventComputerA_MSK: Serial size
  mismatch: Got 1588, Expected 1956"). Steam "Verify integrity" replaced it (6.7 MB);
  the shell menu then loads. Lesson: verify integrity BEFORE copying a test install.
- `-nohomedir` (string present in exe) makes the game write Config/Logs/SaveData into
  `<install>/XComGame/...` instead of `%USERPROFILE%\Documents\My Games\...`
  (generated XCom*.ini appear in `<install>/XComGame/Config`). Keeps tests away from
  the owner's Steam Cloud folder.
- Second instance exits with code 1 before writing a log while the first one runs.
  Cause: mutex `\Sessions\<n>\BaseNamedObjects\UnrealEngine3_<n>` (created via
  `CreateMutexW("UnrealEngine3_%d%s")`). Closing that handle inside instance A
  (DuplicateHandle with DUPLICATE_CLOSE_SOURCE, same user, no admin) lets B start;
  both log "STEAMWORKS initialized 1". Closing `CDdf212806D6EmB31yE0c` and
  `{52813408-...}_0` did not help (not the guard).
- Second install = hard-link clone of the first, but with its own copy of
  `XComGame/Config` (the game rewrites inis there).
- Tools: `tools/launch_pair.ps1`, `close_remote_handle.py`, `hardlink_clone.py`,
  `re_callers.py`, `re_dis.py`, `trace_start.py` (in the owner's workspace, to be moved
  into the game repo).

## Session 1 (cont.) — first network experiments (two instances, one PC)

Mod repo: `WOTCCoopCampaign` (owner's workspace; to be published as its own repo).
Test bot: `CoopHarness` (Actor, steps from `XComCoopHarness.ini`, progress kept on the
class default object across map loads) + `CoopHarnessNet` (Object holding network
delegates). Runner: `tools/run_pair_test.ps1` (activates the mod through the test installs'
`DefaultModOptions.ini`; the game regenerates `XComModOptions.ini` from it).

### Observed
1. `XComGameStateNetworkManager.CreateClient` with a dotted host does not send a single
   packet (UDP sniffer on the target port: 0 packets; B log shows the peer as
   `0.0.0.127`). With the host as ONE decimal integer (127.0.0.1 -> "2130706433") the
   client sends at once (95 packets in 30 s). Applies to `IpNetDriverSteamworks` IP mode.
2. Client and server on one PC must not share the port: the client logs
   "Game client on port 7777" (URL default port). Server on 7778 works.
3. `HasConnections()` is already true while a server only listens; use the
   `PlayerJoined` / `NotifyAcceptedConnection` delegates to detect a real peer.
4. Delegates on the network manager must not point at an Actor: the manager outlives
   maps, the old World is kept alive and the next map load asserts
   ("World XPACK_Shell_Intro.TheWorld not cleaned up by garbage collection!",
   UnGame.cpp:3107, referencer `XComGameStateNetworkManager...ReceiveHistoryDelegates`).
5. Two-way remote commands work (CoopPing/CoopPong both directions).
6. **Strategy history transfer works**: A starts a campaign (cheat start, 225 frames,
   22 soldiers, 4000 supplies), `MPSendHistory`; B in the main menu receives
   `ReceiveHistory frames=225` with the same first soldier (ObjectID 411) and supplies,
   then `open Avenger_Root?game=XComGame.XComHeadQuartersGame` loads A's campaign on B.
   The connection survives the map travel (ping/pong after the load).
7. After loading the Avenger, B's history grows by 7 local frames (225 -> 232) that A
   does not have: map-load side effects create game states locally.

### Derived
- Join flow for the shared campaign: host loads save -> client connects -> host sends
  full history -> client opens the Avenger. Proven end to end in the harness.
- Frames created locally on load (item 7) must either be suppressed on the client or
  the client must accept being overwritten by the host's next partial history.

### Unknown / next
- Steam transport (`steamsockets`) between two different Steam accounts.
- Does SendPartialHistory (delta) apply on top of the client's 7 extra frames?
- How the Avenger/Geoscape visuals refresh when history changes under a loaded map.
