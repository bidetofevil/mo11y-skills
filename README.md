# mo11y-skills

A collection of Claude Code skills for mobile observability.

## Installation

```
/plugin install bidetofevil/mo11y-skills
```

## Available Skills

### `/integrate-embrace`

Integrates the [Embrace Android SDK](https://embrace.io) into any Android project. OpenTelemetry Spans and Logs will be outputted in logcat to verify that integration was successful.

```
/integrate-embrace
```

Run it in the root of any Android project. It handles:
- Kotlin DSL and Groovy build files
- Version catalogs and direct dependency declarations
- Existing or missing Application subclasses
- Core library desugaring for minSdk < 26