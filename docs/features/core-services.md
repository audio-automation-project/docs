# Core Services Ownership

## Scope

This document defines service boundaries and integration contracts between active backend services.

## Service Boundaries

### `audio-library-automation-bot`

- System entrypoint for business workflows.
- Owns orchestration, scheduling, and user-facing automation APIs.
- Calls support services through defined interfaces.

### `audio-silence-service`

- Additional backend service for specialized audio processing tasks.
- Exposes processing endpoints consumed by the core backend.
- Does not own orchestration logic for the overall platform.

## Integration Contract

### Request Contract (Core -> Support)

- Core service sends job metadata and input references.
- Support service returns processing status and output references.

### Error Contract

- Support service returns structured error payloads.
- Core service maps support errors to platform-level responses.

## Responsibility Matrix

| Concern | Core Bot | Silence Service |
|---|---|---|
| API Gateway for platform workflows | Yes | No |
| Platform orchestration | Yes | No |
| Specialized audio processing | No | Yes |
| User-facing operation status | Yes | Partial |
