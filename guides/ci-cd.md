# CI/CD

Continuous integration and deployment workflows for the Audio Library System, managed via GitHub Actions in the `audio-platform-cicd` repository.

## Workflows

The platform uses four GitHub Actions workflows stored in `audio-platform-cicd/.github/workflows/`:

| Workflow | File | Purpose |
|----------|------|---------|
| **Validate Backend** | `validate-backend.yml` | Builds and tests the Java backend |
| **Validate Frontend** | `validate-frontend.yml` | Builds and lints the React frontend |
| **Validate Silence Service** | `validate-silence-service.yml` | Validates the Python ML service |
| **Deploy** | `deploy.yml` | Production deployment with manual approval |

## Validate Backend (`validate-backend.yml`)

Runs on the `audio-library-automation-bot` module:
- Builds with Maven
- Runs unit and integration tests
- Validates compilation and dependency resolution

## Validate Frontend (`validate-frontend.yml`)

Runs on the `audio-frontend` module:
- Installs npm dependencies
- Runs TypeScript type checking (`tsc --noEmit`)
- Builds production bundle (`npm run build`)
- Runs Vitest unit tests

## Validate Silence Service (`validate-silence-service.yml`)

Runs on the `audio-silence-service` module:
- Installs Python dependencies
- Validates imports and module structure

## Deploy (`deploy.yml`)

Production deployment workflow:
- Requires **manual approval** before execution
- Deploys validated artifacts to the target environment

## Environment Configuration

Environment-specific settings are stored in `audio-platform-cicd/environments/`:

```
audio-platform-cicd/
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       ├── validate-backend.yml
│       ├── validate-frontend.yml
│       └── validate-silence-service.yml
└── environments/
    └── example.env
```

## Related Documentation

- [Local Development](local-development.md) — running services locally
- [Configuration](configuration.md) — environment variables
