# Patch: close accepted bridge sockets on broker stop

Base: `@vymalo/opencode-browser` 0.20.2, upstream
`ADORSYS-GIS/lightbridge-opencode-toolbeit` commit
`a35f9511521d93e07cb30f64ce0785af60fb2738`.

## Defect

`Broker.stop()` cleared its registration maps and then called
`BridgeTransport.stop()`, whose Node implementation only called
`server.close()`. Already-accepted WebSocket clients stayed open while their
registrations were gone. A frame from such a client after the stop was rejected
as `browser_frame_unauthenticated`, and `browser_targets` could hang instead of
the guest reconnecting. A plugin reload (the normal path for a config change)
left these stale sockets behind until a full backend restart.

## Fix (one implementation, in the package's own transport)

- `src/node-transport.ts`: `stop()` closes every accepted client from
  `WebSocketServer.clients` with code `1001` and reason `bridge stopped`, then
  closes the listener. Guests see a normal close and reconnect.
- `src/broker.ts`: `stop()` closes the transport before clearing the maps, so a
  frame already queued on a live socket cannot arrive after its registration was
  removed.

No second bridge, no wrapper, no monkey patch: the package's transport is the
only owner of accepted sockets.

## Isolated proof (outside this repository)

A host-side probe started a broker, connected an agent and a Firefox-extension
guest through `hello`/`ready`, called `stop()`, and observed:

- both sockets closed within 1 ms with code `1001`, reason `bridge stopped`;
- `socket_open_after_stop=false`;
- both guests reconnected to a restarted broker and received `ready`;
- no `browser_frame_unauthenticated` and no handshake rejection.

## Build

`node scripts/build-package.mjs .` (the upstream per-file build, unchanged),
15 files and 15 declarations.
