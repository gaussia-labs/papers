# Gaussia Papers

This repository hosts research papers for the [Gaussia](https://github.com/gaussia-labs) AI evaluation framework.

Papers proposed here drive the development of new metrics and implementations across all Gaussia language SDKs ([pygaussia](https://github.com/gaussia-labs/pygaussia), and more to come).

## How It Works

1. **Propose** — Open a Discussion in the [Proposals](https://github.com/gaussia-labs/papers/discussions/categories/proposals) category with your idea
2. **Write** — Draft your paper in LaTeX and open a Pull Request
3. **Review** — The community reviews and discusses your paper in the PR
4. **Accept** — Once approved and merged, the paper is accepted into the Gaussia framework
5. **Implement** — Implementation issues are created in the corresponding SDK repos

## Submitting a Paper

1. Fork this repo
2. Create a folder: `papers/YYYY-MM-your-paper-title/`
3. Use the [LaTeX template](template/) as a starting point
4. Open a PR following the [pull request template](.github/pull_request_template.md)

See [CONTRIBUTING.md](CONTRIBUTING.md) for full details.

## Repository Structure

```
papers/
├── template/               # LaTeX template for new papers
├── papers/                 # Accepted papers
│   └── YYYY-MM-title/      # One folder per paper
│       ├── main.tex
│       ├── references.bib
│       └── figures/
├── CONTRIBUTING.md
└── README.md
```

## Discussion Categories

| Category | Purpose |
|----------|--------|
| **Proposals** | Pitch an idea before writing the full paper |
| **Review** | Formal discussion during paper review |
| **General** | Open questions, meta-discussions, community chat |

## License

All papers are published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
