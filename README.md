# OpenNefia Planning

This repository will contain feature planning information for OpenNefia, organized as a set of RFCs. 

## Motivation

The purpose of this repo is so multiple people contributing to OpenNefia will be on the same page as to how major features will look to players, modders and contributors before any code is written. Since OpenNefia is a large project by its nature, it's difficult to be sure of how a feature will ultimately look to each set of people if there is no single place that collects each maintainer's research and opinions in a visible place.

## Details

For the time being, the planning process will resemble that of [rust-lang/rfcs](https://github.com/rust-lang/rfcs).

RFCs are most desirable for each "major" feature to be added to OpenNefia. The definition of what constitutes a "major" feature is pretty subjective, but as a rule of thumb it includes features like cutscene playback or the quest system.

Of course, since development is still in the very early stages, significant changes to the workflow and the RFC format are to be expected.

## Workflow

1. Copy `Resources/0000-template.md` to `Test/0000-my-feature.md`. The RFC number will be replaced with the pull request ID at a later time.
2. Fill in the RFC, using the template as a guide.
3. Open a pull request with the RFC you added in your branch.
4. Replace the RFC number with the pull request ID that was assigned.
5. Discuss the PR to the satisfaction of the maintainers, before it is merged or rejected.

Unlike Rust's RFC process, the planning process for OpenNefia is generally more lax. This repo is mostly used to get feedback from other maintainers and acts as shared knowledgebank for the things that should be added to OpenNefia at some later point. Think of each RFC as a launching point to guide the implementation of each feature.

The process is pretty ad-hoc right now since there are so few maintainers, so it will be subject to change if things start to pick up.

## License

All contributions to this repo must be licensed under MIT.

The template and general workflow were adapted from [rust-lang/rfcs](https://github.com/rust-lang/rfcs), dual-licensed under MIT/Apache-2.0.
