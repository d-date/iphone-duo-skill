# iPhone Duo Skill

A [Claude Code](https://claude.com/claude-code) skill for adapting apps to Apple's iPhone Duo.

This skill draws on six Tech Talks, the Human Interface Guidelines, Apple Developer Documentation, and two iPhone Duo Group Labs (2026-09-16 and 2026-09-17) as primary sources. API spelling and usage have been checked against public documentation and the official code samples on each session page.

## Contents

| File | Coverage |
|---|---|
| `skills/iphone-duo/SKILL.md` | General principles and guidance on which references to read |
| `references/layout.md` | size class, safe area asymmetry, display corners, reserved regions, arrangement |
| `references/bars.md` | vertical bars, item ordering, axis control, badges, overflow, sheets, disabling vertical bars |
| `references/scenes.md` | hinge, multitasking, multiple windows, scene accessories, camera accessories, Core Motion, Web |
| `references/camera.md` | dual front cameras, camera direction, previews, rotation |
| `references/checklist.md` | Migration and validation for existing apps, and an App Store checklist |

## Installation

### Using npx

```bash
npx skills add d-date/iphone-duo-skill
```

The interactive installer lets you choose a global installation (`~/.claude/skills/`) or a project installation (`.claude/skills/`). Use `npx skills update` to update and `npx skills remove` to remove it.

### Manual installation

```bash
git clone https://github.com/d-date/iphone-duo-skill.git
mkdir -p ~/.claude/skills
cp -r iphone-duo-skill/skills/iphone-duo ~/.claude/skills/
```

For project-specific use, place the skill under `.claude/skills/`.

## Caveats

Many iPhone Duo APIs are documented as iOS 27.1+ Beta. They may change before the final release.

The SwiftUI APIs `onHingeChange`, `toolbarVerticalBehavior(_:)`, and `toolbarVerticalCompressionBehavior(_:)`, which were not found in the documentation when the first edition was written, were all confirmed to be listed as of 2026-09-18. Declarations for every API covered by this guide have been confirmed in the documentation. When implementing, verify them with Xcode completion and on a device or simulator.

## Sources

- [Three steps to make your app shine on iPhone Duo](https://developer.apple.com/iphone-duo/prepare/)
- [Prepare and submit your apps for iPhone Duo (News)](https://developer.apple.com/news/?id=kkphp5qo)
- [A Summary of the iPhone Duo Group Lab (Apple Developer Forums)](https://developer.apple.com/forums/thread/847644)

- Tech Talks: Prepare your app for iPhone Duo / Raise the bar with iPhone Duo / Strike a pose with adaptive layouts on iPhone Duo / Leverage multiple displays and scenes on iPhone Duo / Build a great camera experience for iPhone Duo / Design for iPhone Duo
- Human Interface Guidelines: Designing for iPhone Duo
- Meet with Apple: iPhone Duo Group Lab [2026-09-16](https://www.youtube.com/watch?v=0zp4gAgC6TI) / [2026-09-17](https://www.youtube.com/watch?v=zAaPDbDKvaU)
- Apple Developer Documentation: Preparing your app for iPhone Duo / Registering a camera capture accessory on iPhone Duo / Choosing a camera by the direction it faces / Supporting device rotation in your camera app / iOS 27.1 Beta API reference

The prose in this skill synthesizes the sources above; it does not reproduce Apple's prose or code samples verbatim.

## Changelog

- 2026-10-08: Rewrote the skill in English.
- 2026-10-07: Corrected the guide to state that the simulator also returns reserved regions, and added an overlay to draw them. Incorporated the forum summary of the Group Lab, Apple's prepare guide, Xcode 27.1 beta / RC availability, App Store submission and screenshot requirements, and code correction patterns.
- 2026-09-25: Stated that the iPhone Duo simulator in Xcode 27.1 beta returned zero reserved regions, and added instructions for testing fold avoidance (that statement was incorrect and was corrected on 2026-10-07).
- 2026-09-18: Incorporated answers from the iPhone Duo Group Lab (2026-09-17) and Apple's articles "Preparing your app for iPhone Duo" and "Registering a camera capture accessory on iPhone Duo". Confirmed that the three previously unlisted APIs had been added to the documentation.
- 2026-09-17: Incorporated two Apple camera documentation articles and the iOS 27.1 Beta API reference.
- 2026-09-17: Incorporated answers from the iPhone Duo Group Lab (2026-09-16).
- 2026-09-11: First edition.

## License

MIT License
