# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

homie-mqtt is a Ruby gem implementing the [Homie 4.0.0 IoT convention](https://homieiot.github.io/) for MQTT device publishing. It provides a three-level hierarchy (Device → Node → Property) for describing IoT devices over MQTT.

## Development Commands

```bash
# Install dependencies
bundle install

# Lint
bin/rubocop

# Build the gem
bundle exec rake build

# Install gem locally
bundle exec rake install

# Release to RubyGems
bundle exec rake release
```

There is no test suite in this project. CI only runs Rubocop linting (on Ruby 3.3).

## Architecture

All source lives in `lib/mqtt/homie/`. Entry point is `lib/homie-mqtt.rb` which requires `mqtt/homie`.

### Class Hierarchy

**`MQTT::Homie::Base`** — Abstract base providing `id` (validated against `[a-z0-9][a-z0-9-]*`) and `name` attributes.

**`MQTT::Homie::Device < Base`** — Root entity. Manages MQTT connection, subscription thread, device state (`:init` → `:ready` → `:lost`), and child nodes. Key options: `root_topic:` (default "homie"), `clear_topics:`, `metadata:` (toggle Homie spec metadata publishing).

**`MQTT::Homie::Node < Base`** — Belongs to a Device. Groups related properties. Has a `type` attribute.

**`MQTT::Homie::Property < Base`** — Belongs to a Node. Represents a single value (sensor reading, control). Handles datatype parsing/validation (`:string`, `:integer`, `:float`, `:boolean`, `:enum`, `:color`, `:datetime`, `:duration`), format constraints, retained/non-retained publishing, settable callbacks, and optimistic mode.

### Key Patterns

- **Lazy publishing**: Nothing is sent to MQTT until `device.publish()` is called.
- **Atomic updates**: `device.init { block }` transitions state to `:init`, runs the block, then back to `:ready`. Uses `batch_publish()` internally.
- **Array-like interface**: Device and Node expose `[]`, `each`, `count`, `empty?` for accessing children by ID.
- **`ruby2_keywords`**: Used on `Device#node` and `Node#property` for backward-compatible keyword argument handling across Ruby versions.
- **`MQTT::Homie.escape_id(id)`**: Module-level helper that normalizes arbitrary strings to valid Homie IDs.
- **Metadata control**: `metadata: false` suppresses all `$homie`, `$name`, `$nodes`, `$properties`, etc. topic publishing—only `$state` and values are published.
- **Optimistic mode**: When `optimistic: true` on a property, the value auto-updates to the incoming command after a successful callback.

### MQTT Topic Structure

```
{root_topic}/{device_id}/$state
{root_topic}/{device_id}/{node_id}/{property_id}
{root_topic}/{device_id}/{node_id}/{property_id}/set   # incoming commands (if settable)
```

## Code Style

- Uses `rubocop-inst` (Instructure's Rubocop config) with `rubocop-rake` plugin
- Target Ruby version: 2.5 (must maintain backward compatibility)
- `frozen_string_literal: true` required on all files
- `Style/Documentation` is disabled (no mandatory docstrings)
- Gem dependency on `mqtt-ccutrer` (~> 1.0) for the MQTT client
