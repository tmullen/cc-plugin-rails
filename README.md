# Vanilla Rails Plugin

A Claude Code plugin providing Rails + Hotwire patterns from 37signals.

## Installation

Add to your Claude Code settings:

```json
{
  "plugins": [
    "path/to/vanilla-rails"
  ]
}
```

Or install from a git repository.

## Skills

### vanilla-rails

Use when building Rails applications with Hotwire. Covers:

- **Routing** - REST purity, resources as nouns, nested resources for actions
- **Controllers** - Thin controllers calling rich models, scoping concerns
- **Models** - Small focused concerns, expressive scopes, Current attributes
- **Turbo** - Streams, Frames, broadcasts, array-based DOM IDs
- **Stimulus** - Private methods, event dispatching, lazy loading

### Source

Patterns extracted from 37signals' Fizzy application . Reflects their philosophy of using "pure" Rails conventions without heavy abstractions.

## License

MIT
