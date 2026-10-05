pyenv: cannot rehash: /Users/benjaminkoop/.pyenv/shims isn't writable
# Vocal

Public proof of concept: [github.com/nivoset/vocal-poc](https://github.com/nivoset/vocal-poc).

A single-screen browser proof of concept for checking what a voice assistant understood. Lights and current-weather requests produce previews only. Each timer starts a separate visible countdown in this browser. Timer finish times and named timer presets are saved locally, so countdowns resume after a reload and a named timer can be started again without repeating its duration. Nothing connects to LIFX or a weather service.

## Run

```sh
pnpm install
pnpm run dev
```

Open the local URL Vite prints. Use localhost or HTTPS for microphone access. `pnpm run build` creates the static site in `dist/`; there is no backend. `pnpm run test` checks interpretation and silent-input behavior.

## Deploy to Cloudflare Workers

The Wrangler configuration publishes the Vite build as static assets on a Worker named `vocal`, including SPA fallback for navigation paths. The Needle and Whistle weights are fetched on demand from pinned public Hugging Face revisions, so the deploy command removes the model archives from the Worker assets. The source model files and their checksums remain in the repository. Install dependencies, authenticate with Cloudflare once, then deploy:

```sh
pnpm install
pnpm exec wrangler login
pnpm run deploy
```

`pnpm run deploy` builds the site, removes the weights from the Worker asset directory, and runs `wrangler deploy`. Wrangler prints the deployed `workers.dev` URL. The small runtime JavaScript and WASM files remain in the Worker assets.

Needle and Whistle run in a browser worker using the bundled WASM runtime and pinned model archives on Hugging Face. They are downloaded only when needed. The first typed request receives an immediate preview from a limited local parser while Needle loads. Once ready, Needle makes one pass to classify the action and a second pass over the same message to extract details. If Needle cannot classify a supported request, the limited parser fills in; the interface identifies this fallback. The worker is terminated after two idle minutes or when the page is hidden, allowing its model memory to be reclaimed until the next request. Spoken requests use Whistle to transcribe up to 30 seconds of 16 kHz mono audio, then Needle or the limited parser to interpret the transcript. Audio and text are not posted to a server.

The included model files total about 51 MB, so first load and inference speed vary by device. Releasing the worker frees its runtime memory; browser disk cache can retain the downloaded assets. The header shows a live browser page-memory estimate where supported, or a labeled JavaScript heap estimate in Chromium. These readings are approximate and are not total device RAM; the browser may report `MEMORY N/A` when it exposes neither measure. Record the estimate before and after model load on the target Chromebook to note its observed memory increase.

## Examples

- “Turn on the living room lights”
- “Set the kitchen lights to 40 percent”
- “Set kitchen to fifty percent” (kitchen is in the local light-name list)
- “Ten minutes”
- “Set a pasta timer for 1 minute”
- “Start my pasta timer again” (uses its saved duration)
- “Set a timer for 30 seconds”
- “Set a vacation timer for 2 days and 30 seconds”
- “Set a timer for two weeks”
- “Set a launch timer for tomorrow” (8:00 AM local time)
- “Set a timer for YYYY-MM-DD at 3:30 PM” (replace the date with a future date)
- “Set a timer named pasta for 10 minutes”
- “Cancel my timer” or “Check my timers”
- “What's the weather in Seattle right now?”

Date-only timers default to 8:00 AM in the browser's local time zone. Month-name dates need an explicit year. Missing rooms, durations, years, and locations lead to clarification prompts. Future forecasts and unrelated requests are outside this prototype. Multiple timers can share a name. The most recently started timer with that name supplies the saved preset, and a command to cancel an ambiguous name asks you to choose the instance in the timer cards.

## Third-party models

The runtime and Needle model come from [Cactus-Compute/needle3](https://huggingface.co/Cactus-Compute/needle3), and Whistle comes from [Cactus-Compute/whistle](https://huggingface.co/Cactus-Compute/whistle). Their Apache 2.0 license is copied to `public/models/LICENSE`. Asset SHA-256 checksums are recorded in `public/models/SHA256SUMS`.
