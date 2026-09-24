<div align="center">

# libspc

### Play Super Nintendo `.spc` music files in any browser, with one line of JavaScript.

A pure-JS **SPC700 + S-DSP** emulator and Web Audio player.
No WebAssembly. No build step. No dependencies.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](#license)
![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)
![Platform](https://img.shields.io/badge/platform-browser-orange)
![DSP Rate](https://img.shields.io/badge/DSP-32%20kHz-purple)

</div>

---

## Features

- **SPC700 CPU + S-DSP emulation.** The same sound hardware that powered the Super Famicom / SNES, running entirely in JavaScript.
- **Zero-config link binding.** Any `<a href="song.spc">` on your page becomes a play button automatically.
- **High-quality resampling.** The DSP's native 32 kHz output is converted to your device's sample rate with 4-point Catmull-Rom cubic interpolation, keeping playback smooth and free of aliasing artifacts.
- **Click-free start and stop.** A raised-cosine fade envelope (about 50 ms) eliminates pops when playback begins or ends.
- **Tiny API surface.** `load()`, `loadUrl()`, `play()`, `stop()`. That is all you need.
- **Flexible audio routing.** Pass your own `AudioContext` and destination node to plug the player into an existing Web Audio graph (analysers, gain nodes, effects).
- **Works everywhere.** Exposes itself as `window.SPCPlayer` in the browser and through `module.exports` in CommonJS environments.

---

## Quick Start

### 1. Drop-in link mode (no JavaScript required)

Include the script, then write links:

```html
<script src="libspc.js"></script>

<a href="music/super-mario-world-overworld.spc">Play: Overworld Theme</a>
<a href="music/zelda-lost-woods.spc">Play: Lost Woods</a>
```

Clicking a link downloads, parses, and plays the SPC file immediately. The default click handler matches:

| Selector           | Matches                                 |
| ------------------ | --------------------------------------- |
| `a[href$=".spc"]`  | Links ending in `.spc`                  |
| `a[href*=".spc?"]` | Links with query strings (`.spc?v=2`)   |
| `a[data-spc]`      | Any link explicitly opted in            |

Use `data-spc` for URLs that do not end in `.spc`, such as an API endpoint or CDN route that serves SPC data:

```html
<a href="/api/track/42/download" data-spc>Play track 42</a>
```

### 2. Programmatic mode

```js
const player = new SPCPlayer();

// Download, load, and play in one call
const meta = await player.loadUrl('music/chrono-trigger-corridors-of-time.spc');
console.log(meta);

// Stop with a smooth fade-out
player.stop();
```

### 3. Load from an ArrayBuffer

Useful for drag-and-drop, file inputs, or bundled assets:

```js
input.addEventListener('change', async (e) => {
  const buffer = await e.target.files[0].arrayBuffer();

  const player = SPCPlayer.getInstance();
  const meta = player.load(buffer); // parse and reset emulator state
  player.play();
});
```

---

## API Reference

### `new SPCPlayer([audioCtx], [destination])`

Creates a new player instance.

| Parameter     | Type           | Default                | Description                               |
| ------------- | -------------- | ---------------------- | ----------------------------------------- |
| `audioCtx`    | `AudioContext` | A new `AudioContext`   | Reuse an existing context.                |
| `destination` | `AudioNode`    | `audioCtx.destination` | Where the player's output is connected.   |

```js
// Route SPC audio through your own gain and analyser chain
const ctx = new AudioContext();
const gain = ctx.createGain();
const analyser = ctx.createAnalyser();
gain.connect(analyser).connect(ctx.destination);

const player = new SPCPlayer(ctx, gain);
```

---

### `player.load(buffer)` returns `meta`

Parses an SPC file and resets the emulator. Does not start playback.

- **`buffer`**: `ArrayBuffer` or `Uint8Array` containing the `.spc` file.
- **Returns**: the parsed metadata object, also stored in `player.currentMeta`.

---

### `await player.loadUrl(url)` returns `Promise<meta>`

Fetches an SPC file over HTTP(S), loads it, and starts playback automatically.

- Throws if the request fails (`!response.ok`).
- Subject to standard browser CORS rules. Cross-origin hosts must send `Access-Control-Allow-Origin`.

---

### `player.play()`

Starts or resumes playback with a smooth fade-in. Automatically resumes a suspended `AudioContext`, so it is safe to call from a click handler to satisfy browser autoplay policies.

---

### `player.stop()`

Fades out over roughly 50 ms, then disconnects the audio node to free up CPU.

---

### `SPCPlayer.getInstance()` returns `SPCPlayer`

Returns a lazily created shared singleton. Use it when you only need one player on the page. Link binding uses it internally, so a new click always replaces whatever is currently playing.

---

### `SPCPlayer.bindLinks([selector])`

Attaches a single delegated `click` listener to `document` that turns matching links into SPC play buttons.

```js
// Custom selector: only links inside a playlist container
SPCPlayer.bindLinks('#playlist a');
```

`bindLinks()` is called automatically with the default selector once the DOM is ready. Call it manually only if you want to customize the selector.

---

### Properties

| Property             | Type        | Description                                          |
| -------------------- | ----------- | ---------------------------------------------------- |
| `player.playing`     | `boolean`   | `true` while audio is being generated.               |
| `player.currentMeta` | `object`    | Metadata of the last loaded SPC (title, game, etc.). |
| `player.engine`      | `SPCEngine` | Direct access to the underlying emulator core.       |

---

## How It Works

```
 +-----------+    +--------------+    +--------------+    +--------------+
 |  .spc     | -> |  parseSPC()  | -> |  SPCEngine   | -> |  32 kHz PCM  |
 |  file     |    |  RAM / regs  |    |  SPC700 CPU  |    |  stereo      |
 |           |    |  DSP state   |    |  + S-DSP     |    |  samples     |
 +-----------+    +--------------+    +--------------+    +------+-------+
                                                                 |
                                        +------------------------v-------+
                                        |  Cubic (Catmull-Rom) resampler |
                                        |  32 kHz -> device sample rate  |
                                        +------------------------+-------+
                                                                 |
                                        +------------------------v-------+
                                        |  Raised-cosine fade envelope   |
                                        |  -> ScriptProcessorNode        |
                                        |  -> speakers                   |
                                        +--------------------------------+
```

1. **`parseSPC`** reads the `.spc` snapshot: 64 KB of APU RAM, CPU registers, DSP registers, and ID666 metadata.
2. **`SPCEngine`** restores that state and runs the SPC700 CPU alongside the S-DSP, producing one stereo sample at a time at the native 32,000 Hz.
3. A **4-tap cubic interpolator** resamples the stream on the fly to match `AudioContext.sampleRate` (44.1 kHz, 48 kHz, and so on).
4. A **cosine-shaped gain ramp** smooths every start and stop.
5. Audio is delivered through a `ScriptProcessorNode` in 8192-sample blocks.

---

## Browser Support

Any modern browser with the Web Audio API.

| Chrome | Firefox | Safari | Edge |
| :----: | :-----: | :----: | :--: |
|  Yes   |   Yes   |  Yes   | Yes  |

Safari's prefixed `webkitAudioContext` is handled automatically.

**Autoplay policy:** browsers require a user gesture before audio can start. Link-click binding satisfies this by design. If you call `play()` on page load, it stays silent until the user interacts with the page.

---

## Project Layout

```
libspc.js     SPCPlayer class and link-binding glue
              (requires SPCEngine, SPC700, DSP and parseSPC to be
               loaded or bundled in the same scope)
```

`SPCPlayer` is the playback front-end. It depends on the emulator core (`SPC700`, `DSP`, `SPCEngine`) and the `parseSPC` loader, so make sure these are included before `SPCPlayer` is instantiated.

---

## Use Cases

- Retro game music jukeboxes and playlists
- Game preservation and VGM archive sites
- Web games that want authentic SNES audio without re-encoding to MP3 or OGG
- Visualizers, by routing output through an `AnalyserNode` via the `destination` parameter
- Chiptune and emulation research and education

---

## Notes and Limitations

- Uses `ScriptProcessorNode` for maximum compatibility. It runs on the main thread, so very heavy pages may glitch. Migrating to `AudioWorklet` is a natural next step.
- Only one track plays per player instance. Loading a new file replaces the current one.
- SPC files contain copyrighted game music. Make sure you have the right to host and distribute any tracks you serve.

---

## Roadmap

- [ ] `AudioWorklet` backend
- [ ] Volume, per-voice mute and solo controls
- [ ] Pause and seek support
- [ ] Track length, loop, and fade-out handling from ID666 tags
- [ ] Bundled TypeScript typings

---

## Contributing

Issues and pull requests are welcome. If you are fixing an accuracy bug in the CPU or DSP, including the game and track that reproduces it makes review much faster.

---

## License

Released under the MIT License. See `LICENSE` for details.

---

<div align="center">

*Built for the golden age of 16-bit audio.*

</div>
