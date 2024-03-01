# Snapcast MPRIS wrapper

([MPRIS: Media Player Remote Interfacing Specification](https://specifications.freedesktop.org/mpris-spec/2.2/))

This project is a wrapper around the Snapclient binary and the Snapserver RPC API. 
It exposes the current playing state on the desktop bus (D-BUS), and allows control 
through the DBUS play/pause/stop signals. 

## General working
When the main script (`snapcastmpris.py`) is executed, it will hook into the communication 
busses and set up and endless loop. An instance of `SnapcastWrapper`, which lives 
in a separate thread, is created.

`SnapcastWrapper` has methods to start/pause/play snapcast audio, and keeps track of the _snapclient_ 
process. It gets help from `SnapcastRpcWrapper`, which communicates with _snapserver_, the snapcast server.
This communication with [the Snapserver RPC API](https://github.com/badaix/snapcast/blob/master/doc/json_rpc_api/v2_0_0.md)
is used to control the snapcast audio level, mute status, client name. The `SnapcastRpcWebsocketWrapper` is used to receive 
events from [the Snapserver RPC API](https://github.com/badaix/snapcast/blob/master/doc/json_rpc_api/v2_0_0.md). It receives 
information about the stream that is currently playing, such as the playing state and the name, and passes this on to 
the `SnapcastWrapper`. Volume level changes, muting of a client, client connects, disconnects, ... is handled by this 
websocket wrapper as well. `SnapcastMPRISInterface` is used to communicate through the DBUS, to receive play/pause/stop signals from the OS and to relay information about the current state back to the OS. 

## What SnapcastWrapper does
SnapcastWrapper runs in a separate thread from the main script.
SnapcastWrapper implements the SnapcastRpcListener class and methods, which are called by SnapcastRpcWebsocketWrapper.
When ALSA <=> Snapclient volume synchronisation is enabled, SnapcastWrapper will monitor the ALSA volume level and send
any changes to Snapserver through SnapcastRPCWrapper. Volume synchronisation is disabled by default and enabled through the `--sync_alsa_volume` or `-s` flag or via the configuration with `sync-alsa-volume = 1`. It requires the `alsaaudio` library to be installed. Also the ALS mixer can be provided through the `-m` or `--mixer` flag or via the configuration with `alsa-mixer = Softvol`.

### Playing audio
When playing audio
- A Snapclient process is started 
- Snapclient is unmuted through an RPC call
- Other audio players are stopped (through the HifiBerryOS pause-all script)
- DBUS status is updated

### Pausing audio
When pausing audio
- Snapclient is muted through an RPC call
- depending on the "pause" source, a flag in order to prevent snapclient to switch back to playing on the next stream update. 
It will only switch back to playing automatically when the snapcast source stream has been paused.
- Snapclient keeps running
- DBUS status is updated

### Stopping audio
When stopping audio
- The snapclient process is stopped
- DBUS information is updated
- SnapcastWrapper keeps running in order to act should a play signal come from DBUS or snapserver.

## What SnapcastRpcWrapper does
SnapcastRpcWrapper is a helper class to SnapcastWrapper, and provides access to 
[the Snapserver RPC API](https://github.com/badaix/snapcast/blob/master/doc/json_rpc_api/v2_0_0.md). It can mute and 
unmute the client, set the client volume, change the client latency and change the client name. Through
SnapcastRpcWrapper, the client information and server information can be obtained.

*The advantage of muting* is that snapserver can be configured not to send data to muted clients. This means that a 
muted client will reduce network traffic, compared to a running process with ignored audio.

## What SnapcastRpcWebsocketWrapper does
SnapcastRpcWebsocketWrapper runs a websocket in a separate thread, and calls callback methods in SnapcastWrapper (a SnapcastRpcListener 
implementation) to act on stream and client status changes. 

- When muted, the status is set to paused
- When unmuted, nothing is done. If a stream is already playing to the active device, and the player is in PAUSED mode, 
the player should switch to playing. This has not been implemented yet.
- When a stream switches from idle to playing, and the previous pause event was not caused by a DBUS event, SnapcastWrapper switches to the playing state.
- When a stream switches from playing to idle, the SnapcastWrapper pause logic is triggered to mute the client and switch to the PAUSED state.
- When the snapclient volume level is changed, and ALSA <=> Snapclient volume synchronisation is enabled, the ALSA volume is adjusted.