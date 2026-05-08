# ADR-001: Use JavaCV Instead of FFmpeg CLI

**Date:** 2026-05-05  
**Status:** Accepted  
**Deciders:** Platform team

---

## Context

All audio and video processing in `audio-library-automation-bot` currently runs via `ProcessBuilder` — Java spawns a shell process running the `ffmpeg` binary. This works on a developer machine but creates a hard external dependency: FFmpeg must be installed in every environment where the service runs.

When deploying to **Docker** or **Railway** (PaaS), this means:
- The Dockerfile must include `RUN apt-get install -y ffmpeg`
- FFmpeg version is tied to what the base image's package manager provides
- On Railway, the build environment must have FFmpeg available at a known path
- Local Windows dev requires a separate FFmpeg installation at a system PATH location

Any mismatch in FFmpeg version between environments causes subtle differences in audio/video output — different default codecs, bitrate handling, or container formats.

Additionally, the current `AudioConcatenatorService` writes a temporary `filePath.txt` to build the FFmpeg concat input — a file that is hardcoded and shared across concurrent requests, causing data corruption when two cycles run simultaneously.

---

## Decision

Replace all `ProcessBuilder`-based FFmpeg calls with **JavaCV** (Bytedeco FFmpeg presets via JavaCPP).

JavaCV bundles the FFmpeg native shared libraries (`.so` on Linux, `.dll` on Windows) as Maven JAR dependencies with platform classifiers. At startup, JavaCPP extracts and loads the native libraries from the classpath. No external binary installation is required.

**Maven dependency approach (in `foundation/pom.xml`):**

```xml
<dependency>
    <groupId>org.bytedeco</groupId>
    <artifactId>javacv</artifactId>
    <version>1.5.10</version>
</dependency>
<!-- Platform classifier set by Maven profile -->
<dependency>
    <groupId>org.bytedeco</groupId>
    <artifactId>ffmpeg</artifactId>
    <version>6.1.1-1.5.10</version>
    <classifier>${ffmpeg.classifier}</classifier>
</dependency>
```

Maven profiles set `ffmpeg.classifier` to `linux-x86_64` on CI/Docker/Railway and `windows-x86_64` on developer machines. This keeps JAR size down — we only bundle the binary for the target platform, not all platforms.

---

## Consequences

### Positive
- **Zero container dependency** — Dockerfile is `FROM eclipse-temurin:17-jre-alpine` only, no apt-get
- **Pinned FFmpeg version** — `6.1.1` is in `pom.xml`, consistent across all environments
- **Eliminates the `filePath.txt` concurrent-write bug** — JavaCV concatenation streams frames directly, no temp file needed
- **Real progress tracking** — frame-level callbacks, not "wait for process exit"
- **Proper Java exceptions** — not exit code + stderr string parsing
- **Exact duration** — `grabber.getLengthInTime()` is reliable for VBR MP3s; current ffprobe stderr parsing is not
- **Railway deploy works with `git push`** — no environment setup

### Negative
- **Larger JAR** — adds ~80–120MB for the Linux x86_64 FFmpeg native libraries. Mitigated by using classifiers (not `-platform` which adds all OS variants)
- **More complex API** — JavaCV mirrors the C API; it is lower-level than CLI flags. Mitigated by encapsulating all operations in `FFmpegOperations` in `foundation` — services call high-level methods, not JavaCV directly
- **GPU encoding requires explicit codec selection** — `h264_nvenc` for GPU, `AV_CODEC_ID_H264` for CPU. Railway standard plans are CPU-only, so CPU encoding is the default; GPU is opt-in via env var

### Neutral
- Existing FFmpeg CLI knowledge is still useful for understanding what JavaCV is doing internally — the concepts (codecs, muxers, filters) are identical

---

## Alternatives Considered

**Keep `ProcessBuilder` + ensure FFmpeg in Docker image**
Rejected. The problem is not just Docker — it is Railway's build environment variability and the inability to pin the exact FFmpeg version without building a custom base image. The concurrent `filePath.txt` bug also remains unsolved.

**Use `javacv-platform` (all platforms in one dependency)**
Rejected. Pulls ~500MB of native binaries for Windows, Mac, Linux, ARM, Android. The platform-classifier approach is strictly better.

**Use mp3agic / JLayer for audio metadata only, keep ProcessBuilder for encoding**
Rejected. These libraries are unreliable with VBR MP3s and files with embedded cover art. The whole point is to eliminate the patchwork of workarounds.

---

## Implementation Location

**Implemented in the monolith:** `audio-library-automation-bot/src/main/java/kg/automation/rest/automatation/ffmpeg/FFmpegOperations.java` (single Spring bean; services inject `FFmpegOperations` and do not use JavaCV types directly). A future **`foundation`** submodule may extract this package unchanged.

This is the single shared bean. Every service that needs audio/video operations injects `FFmpegOperations`. No service calls JavaCV directly.
