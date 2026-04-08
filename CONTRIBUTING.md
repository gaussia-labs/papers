# Contributing to Gaussia Papers

## Paper Submission Process

### 1. Propose Your Idea

Before writing a full paper, open a Discussion in the **Proposals** category:

- Describe the metric, algorithm, or evaluation approach
- Explain what problem it solves
- Reference any related work
- Indicate which SDKs would benefit (Python, TypeScript, etc.)

This lets the community give early feedback before you invest time in a full paper.

### 2. Write Your Paper

- Fork this repository
- Copy `template/` to `papers/YYYY-MM-your-paper-title/`
- Write your paper in LaTeX using the provided template
- Include all figures in the `figures/` subdirectory
- Add references to `references.bib`

### 3. Open a Pull Request

- Target the `main` branch
- Fill out the PR template completely
- Link the original Proposal discussion if one exists
- Ensure the paper compiles without errors

### 4. Review Process

- At least **2 community reviewers** should approve
- Reviewers will evaluate:
  - **Novelty**: Does this add something new to AI evaluation?
  - **Soundness**: Is the methodology rigorous?
  - **Clarity**: Is the paper well-written and reproducible?
  - **Feasibility**: Can this be implemented in the Gaussia framework?
- Authors address feedback by pushing updates to the PR

### 5. After Acceptance

Once merged:
- The paper is officially part of the Gaussia framework
- Implementation issues are created in the relevant SDK repos
- The paper authors are credited in the implementation

## Paper Format

- **Language**: English
- **Format**: LaTeX (use the provided template)
- **Length**: No strict limit, but be concise
- **Sections** (recommended):
  1. Abstract
  2. Introduction & Motivation
  3. Related Work
  4. Methodology
  5. Implementation Considerations
  6. Evaluation / Experiments (if applicable)
  7. Conclusion
  8. References

## Code of Conduct

- Be constructive in reviews
- Critique ideas, not people
- Welcome newcomers
- Give credit where due
