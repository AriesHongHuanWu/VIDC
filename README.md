# VIDC

A browser video chat room. Media travels peer to peer over WebRTC; Firebase Firestore is used only as the signalling channel and presence list. There is no media server.

**Live demo: https://arieshonghuanwu.github.io/VIDC/**

Open the link, create a room or join the public lobby, and share the resulting `?room=<id>` URL. Camera and microphone permission is required. The interface text is Traditional Chinese.

## Topology

Full mesh. Every participant opens an `RTCPeerConnection` to every other participant, so a room of `n` people holds `n(n-1)/2` connections and each client encodes and uploads `n-1` outbound streams. That is why the room cap is a hard 8, enforced on join by counting the room's user documents with `getDocs` before writing your own.

Mesh was the right call here because it needs no infrastructure: no SFU to run, no per-minute cost, and no server that can see the media. It is also the reason the design stops at 8 rather than scaling further.

## Signalling over Firestore

Two collections per room under `artifacts/<appId>/public/data/webrtc_liquid_chat/<roomId>/`:

- `users` is the presence list. Joining writes a document keyed by the Firebase anonymous auth UID with a nickname and a server timestamp. A snapshot listener turns `added` / `modified` / `removed` document events directly into peer connection setup, nickname updates and teardown.
- `signals` is an ephemeral mailbox. Each SDP offer, answer and ICE candidate is one document tagged `{ to, from }`. Every client subscribes with `where("to", "==", userId)` and deletes each document as soon as it is consumed, so the collection stays near empty instead of growing into a transcript of the session.

Identity is Firebase anonymous auth, and the UID doubles as the peer ID. `joinRoom` awaits `onAuthStateChanged` before touching Firestore so nothing races the sign-in.

## Audio pipeline

The microphone track is not sent to peers directly. It is routed through a WebAudio graph first:

```
getUserMedia -> MediaStreamSource -> GainNode -> AnalyserNode -> MediaStreamDestination -> RTCPeerConnection
```

That indirection buys three things:

- **Input gain.** A slider drives `gainNode.gain`, so a quiet microphone can be boosted up to 2x before transmission rather than at the listener's end.
- **A software noise gate.** The analyser's average magnitude is compared against a user-set threshold every animation frame, and gain is dropped to zero below it. Because the gate sits before the destination node, gated audio is never encoded or sent.
- **Speaking detection.** The same analyser reading drives a per-tile speaking indicator, with a 500 ms timeout so the ring does not flicker between syllables. Remote tiles get their own `AudioContext` and analyser built from the received stream.

The browser's own `noiseSuppression` and `echoCancellation` constraints are applied on top and can be toggled live with `applyConstraints`, falling back to reacquiring the stream if the browser rejects the change.

Aggregate speech volume also drives the background: the CSS blob animations are picked up through `element.getAnimations()` and their playback rate is scaled between 0.5x and 3x by the current audio level, which decays 5% per frame. The background moves when the room is loud and settles when it is quiet.

## Devices, layout and rooms

- Input devices are enumerated with `enumerateDevices` and can be switched mid-call. Switching reacquires the stream and calls `replaceTrack` on every existing sender, so the renegotiation cost of a new offer is avoided.
- The video grid picks one of seven CSS layouts based on participant count, with distinct treatments for 1, 2, 3, 4, 5-6, 7-9 and 10+ tiles.
- Room ID is reflected into the URL with `history.pushState` and read back from `?room=` on load, so a shared link joins directly.
- Leaving closes every peer connection, stops all tracks, closes the `AudioContext`, and deletes the presence document. The same delete is registered on `beforeunload`.

## Architecture

One file, `index.html`. No build step and no package manifest. Firebase v11.6.1 is imported as ES modules from `gstatic.com`, and Tailwind comes from its CDN. A GitHub Actions workflow publishes the repository root to GitHub Pages on every push to `main`.

Running your own copy requires your own Firebase project, since the config block in the file points at the author's. You need Firestore enabled and anonymous sign-in turned on, plus rules that permit reads and writes under the `artifacts/<appId>/public/data/**` path.

## Known limitations

These are properties of the current code, not a roadmap:

- **STUN only.** The ICE configuration lists Google's public STUN servers and no TURN server. Peers behind symmetric NAT or a restrictive corporate firewall will fail to connect, and there is no relay fallback.
- **Simultaneous offers on join.** A joining client sees existing participants as `added` and offers to them, while those participants see the newcomer as `added` and offer back. There is no perfect-negotiation rollback (no polite/impolite peer role), so the first handshake can collide.
- **Presence relies on `beforeunload`.** A crashed tab or a killed browser leaves a stale user document, which counts against the 8-person cap until it is removed manually. There is no TTL or server-side cleanup.
- **No access control.** Anyone with the room ID can join. Rooms have no owner, no lock and no kick.

## What this is not

Not a Zoom or Meet replacement. There is no screen sharing, no text chat, no recording, no virtual background, no transcript, and no persistence of anything beyond the nickname in `localStorage`. Media is encrypted in transit because WebRTC mandates DTLS-SRTP, not because of anything this project adds.

It is a personal project written to understand WebRTC signalling end to end without hiding it behind an SDK.

## License

MIT. See [LICENSE](LICENSE).
