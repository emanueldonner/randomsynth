# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- Mixer component with volume, pan, and reverb controls.
- ModuleContainer component for channel UI and layout.
- Synth component integrating with Tone.js and supporting multiple synthesizer configurations.
- SynthComponent for individual synth controls (waveform, octave range).
- Global reverb bus for shared audio effects.

### Changed

- Improved synth initialization logic to avoid overwriting mixer values.
- Updated mixer logic so reverb send settings sync correctly with configuration.
