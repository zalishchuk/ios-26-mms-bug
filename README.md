# iOS 26 ManagedMediaSource Cold Start Bug

**Reproduction Demo:** https://zalishchuk.github.io/ios-26-mms-bug/

On iOS 26 Safari, when the browser is "cold started" (force-closed and reopened), concurrent `appendBuffer` calls fail due to a race condition in `mediaparserd` initialization.

> [!IMPORTANT]
> See [INVESTIGATION.md](./INVESTIGATION.md) for detailed iOS system logs and analysis of the `mediaparserd` race condition.

https://github.com/user-attachments/assets/0f4c5b88-652f-4c82-8ccc-e6312c07f41e

## The Bug

- `readyState` shows `"open"` but internally it's `"ended"`
- `sourceended` event never fires
- Only affects the **first** MMS after cold start
- Reload or new tab works fine

### Affected Versions

| Environment                | iOS 26.0 | iOS 26.1 | iOS 26.2 |
| -------------------------- | -------- | -------- | -------- |
| Xcode Simulator            | OK       | OK       | OK       |
| Real device (BrowserStack) | OK       | OK       | BUG      |
| Real device (physical)     | ?        | ?        | BUG      |

> [!NOTE]
> The bug cannot be reproduced on Xcode-emulated iOS 26.2 devices, but still reproduces on real physical devices.

## Reproduction

1. Force-close Safari from the app switcher
2. Reopen Safari and load the test page
3. The "simultaneous" test fails with `readyState=ended`

## Test Cases

| Test                     | Description                     | Cold Start         | After Reload |
| ------------------------ | ------------------------------- | ------------------ | ------------ |
| `simultaneous`           | Append to 2 SBs at once         | FAIL               | PASS         |
| `sequential`             | Append one at a time            | PASS               | PASS         |
| `single`                 | Only 1 SourceBuffer             | PASS               | PASS         |
| `warmup`                 | Warmup MMS first                | PASS               | PASS         |
| `twice`                  | Two attempts back-to-back       | 1st FAIL, 2nd PASS | PASS         |
| `dual-mms`               | 2 MMS, 1 SB each, concurrent    | FAIL               | PASS         |
| `dual-mms-single-append` | 2 MMS, 1 SB each, single append | PASS               | PASS         |

## Root Cause

Concurrent parsing requests to `mediaparserd` during its first connection to the GPU process after browser cold start. The race is not limited to multiple SourceBuffers within one MMS — even two separate MMS instances with one SourceBuffer each will fail if they append simultaneously. One request succeeds, the other fails with `ParsingFailed`.

## Workarounds

1. **Recovery**: Detect the error and create a fresh MMS
2. **Warmup**: Create a sacrificial MMS, complete one append, then use the real MMS
3. **Sequential**: Serialize the first append (wait for `updateend` before second append)

## Related

- https://webkit.org/blog/17333/webkit-features-in-safari-26-0/#media
- https://github.com/video-dev/hls.js/pull/7683
- https://github.com/video-dev/hls.js/pull/7693
- https://github.com/video-dev/hls.js/issues/7687
