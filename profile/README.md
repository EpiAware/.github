# EpiAware

A composable Julia ecosystem for infectious disease modelling.

## Why

Infectious disease models that integrate multiple data sources provide better evidence for outbreak response than chains of separate models, but building them is slow and requires expertise across domains.
Composable modelling, where validated components combine into joint models that properly propagate uncertainty, addresses this but requires an ecosystem of reusable infectious disease model components.
We believe Julia is the best language for this ecosystem due to its type system, multiple dispatch, automatic differentiation support, and existing scientific computing infrastructure ([SciML](https://sciml.ai), [Turing.jl](https://turinglang.org), [Distributions.jl](https://github.com/JuliaStats/Distributions.jl)), which provide the foundations composable modelling needs.

## Background

In R, we have built the [epinowcast](https://github.com/epinowcast) ecosystem (packages, community forum, seminar series) and developed several other widely used packages including [EpiNow2](https://github.com/epiforecasts/EpiNow2) and [scoringutils](https://github.com/epiforecasts/scoringutils).
These are used by infectious disease academics and public health departments internationally.
We want to create something equivalent in Julia: a domain-focused ecosystem in the mould of [SciML](https://sciml.ai) or [Turing.jl](https://turinglang.org), with the community infrastructure of [rOpenSci](https://ropensci.org) and the domain specificity of [SpeedyWeather.jl](https://github.com/SpeedyWeather/SpeedyWeather.jl).

## Packages

Package documentation is collected on the [ecosystem site](https://epiaware.org) and its [unified docs browser](https://epiaware.org/docs.html).
Most packages extend [Distributions.jl](https://github.com/JuliaStats/Distributions.jl) so they share an interface and compose cleanly.
Empty scaffold repositories are listed as *planned* until they have real, documented functionality.

### Active

| Package | Description |
|---|---|
| [CensoredDistributions.jl](https://github.com/EpiAware/CensoredDistributions.jl) | Primary, interval, and double interval censoring for epidemiological delay distributions |
| [ComposedDistributions.jl](https://github.com/EpiAware/ComposedDistributions.jl) | A verb grammar for n-ary composition over any Distributions.jl distribution |
| [ConvolvedDistributions.jl](https://github.com/EpiAware/ConvolvedDistributions.jl) | Distribution convolution and shared numerical quadrature for Distributions.jl |
| [ModifiedDistributions.jl](https://github.com/EpiAware/ModifiedDistributions.jl) | Wrappers that each change one behaviour of a distribution — rescaling, likelihood weighting, hazards, or transforms |
| [ReparameterisedDistributions.jl](https://github.com/EpiAware/ReparameterisedDistributions.jl) | Alternative parameterisations for Distributions.jl |
| [EpiAwarePackageTools.jl](https://github.com/EpiAware/EpiAwarePackageTools.jl) | Shared CI, documentation, quality, and AD-benchmark tooling for the ecosystem |
| [EpiAwareR](https://github.com/sbfnk/EpiAwareR) | R interface to EpiAware.jl (prototype) |

### In development

| Package | Description |
|---|---|
| [ComposableTuringIDModels.jl](https://github.com/EpiAware/ComposableTuringIDModels.jl) | Composable probabilistic infectious disease modelling built on Turing.jl — the ecosystem prototype |

### Planned

| Package | Description |
|---|---|
| [GenerationTime.jl](https://github.com/EpiAware/GenerationTime.jl) | Representing and estimating generation time distributions |
| [DEdiseasecomponents.jl](https://github.com/EpiAware/DEdiseasecomponents.jl) | Reusable components for differential equation infectious disease models |

### Papers

| Repo | Description |
|---|---|
| [ComposableProbabilisticIDModels](https://github.com/EpiAware/ComposableProbabilisticIDModels) | The case for composable probabilistic infectious disease models |
| [JuliaForIDM](https://github.com/EpiAware/JuliaForIDM) | Julia for applied infectious disease modelling |

## Contributing

We are at an early stage and actively looking for collaborators.
If you are interested in composable modelling, infectious disease epidemiology, or Julia ecosystem development, please open an issue or get in touch.
