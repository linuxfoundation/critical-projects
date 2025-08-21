# Linux Foundation Critical Projects

[![License](https://img.shields.io/badge/License-CC%20BY%204.0-blue)](https://creativecommons.org/licenses/by/4.0/)
![Version](https://img.shields.io/badge/version-0.1-green)
![Author](https://img.shields.io/badge/author-Sam_Boysel-purple)

A methodology for identifying and ranking open source software projects in terms of *criticality*.

**Contents**

- [Objective](#objective)
    - [What does it mean for a project to be "critical"?](#what-does-it-mean-for-a-project-to-be-critical)
- [Methodology](#methodology)
    1. [Project discovery and filtering](#project-discovery-and-filtering)
    2. [Measure signals of criticality](#measure-signals-of-criticality)
    3. [Score the set of gathered projects](#score-the-set-of-gathered-projects)
        - [Signal and parameter choices](#signal-and-parameter-choices)
- [Limitations](#limitations)
- [Future iterations](#future-iterations)
- [References](#references)

# Objective

Using the [OpenSSF Criticality Scores](https://openssf.org/projects/criticality-score/) (hereafter, **OpenSSF CS**) as a foundation, develop a scoring system that ranks open source software projects in terms of "criticality" or "importance". The system ought to

1.  Identify a set of "top projects" and provide relative criticality ranks for this set
2.  Address some limitations identified by the OpenSSF CS
    - Reduce the prevalence of "false positives"
    - Use a set of signals that be consistently measured across projects
    - Fine-tune to the scoring function and signal weights
3.  Be easily deployed
4.  Be easily iterated upon to improve performance

## What does it mean for a project to be "critical"?

An open source project is "critical" if

1.  it is either directly relied on by many users or indirectly through other widely-used projects that depend on it.
2.  in the event that a critical security vulnerability were found and exploited, it could cause catastrophic damage (economic, societal, human life, safety, etc.).

# Methodology

Our aproach consists of three main steps:

1. [Project discovery and filtering](#project-discovery-and-filtering)
2. [Measure signals of criticality](#measure-signals-of-criticality)
3. [Score the set of gathered projects](#score-the-set-of-gathered-projects)

## Project discovery and filtering

The universe of open source projects is too broad, so we need to pare this down a bit before even beginning to rank. Let's define the unit of analysis for an open source project as the project's canonical source code repository. Our approach to project discovery and filtering is as follows:

1. Use git-based projects with public source repositories on GitHub
2. Use GitHub GraphQL to return a list of the top 10,000 most starred public repositories older than 6 months. We additionally include the OpenSSF Securing Critical Projects list of "most critical projects" to this set.
3. Filter this set to exclude forks, mirrors, templates, archived repositories.

## Measure signals of criticality

For each project in the set of  projects, we next gather a number of quantitivative signals that can be derived from project characteristics. Signals should measure something about the project that reasonably correlates with a notion of criticality.

**Examples:**

* Project age
* Development and release frequency
* Number of distinct individual contributors
* Number of distinct contribution organizations
* Level of project discourse (contributor communication, responsiveness to issues)
* Merge/pull requests, forks, artifact downloads
* Downstream dependents
* Scope of use (*who* and *how* is the project being used?)

Some of these signals are easier to measure than others. For example, project age can be easily measured by looking at the date of the first commit in the project's version control system (VCS). On the other hand, scope of use is much harder to measure, as it requires knowledge of who is using the project and how they are using it.

If signals are measured inconsistently across projects, including the signal in a criticality score can bias rankings. For example, if we were to use "number of stars" as a signal, this would be problematic because not all projects are hosted on GitHub. Even among those that are, the number of stars can vary widely depending on the project's popularity and the community's engagement. A concern with the OpenSSF CS data is that some signals like dependency information and issue activity are not consistently measured across projects.

We focus on signals that can be consistently measured across all projects in our set. The signals we have chosen so far are **based purely on the version control log** of the open source project. Our hope is to expand beyond this constaint in future iterations.

## Score the set of gathered projects

Project criticality rankings can be determined by a composite score derived from the project signals. The ideal score should be a near perfect correlate of criticality: the higher the score, the more "critical" the project is. For each project $i$, the project's criticality score is a function $f$ of the $K$ different criticality signals $s_{ik}, \ldots, s_{iK}$ and parameters $\theta$:

$$
score_{i} = f(s_{i1}, \ldots, s_{iK}; \theta)
$$

Note that the function $f$ could take many forms. Following the OpenSSF CS, we opt for a scoring function that is a linear combination of signals and parameters:

$$
score_{i} = \sum_{k=1}^{K} \alpha_{k} g(s_{ik}) \
$$

where

| Term | Description |
| --- | --- |
| $g(s_{ik})$ | $g$ is a function of the numerical signal $s_{ik}$. Our preferred approach is to define $g$ as the **percentile rank of project's signal** relative to all other projects. For example, if project has more contributors than 75% of other projects in the set of projects, then = 0.75. The original OpenSSF CS uses a clamping function for $g$ dependent parameterized thresholds to map the signals to a common scale. The percentile rank transformation is attractive choice for $g$ since it scales all signals to the $[0, 1]$ interval, is directly interpretable, and requires fewer parametric assumptions than the OpenSSF CS implementation. The drawback is that it can only be calculated when the signal has been measured for *all* projects under consideration.|
| $\alpha_{k}$ | Relative weights for signal $k$. Larger values for $\alpha_{k}$ mean signal $k$ matters more for determining the score. Assume $0 \leq \alpha_{k} \leq 1$ and $\sum_{k} \alpha_{k} = 1$|

Under these assumptions, **the score is a weighted composite of signal ranks**.

### Signal and parameter choices


| Weight $\alpha_{k}$ | Signal $s_{ik}$ | Description |
| --- | --- | --- |
| 50% | Distinct contributors | Numer of distinct committers |
| 20% | Distinct organizations | Number of distinct organizations contributing to the project, as determined by email domain |
| 10% | Project size | Count of source lines of code in the project |
| 10% | Last updated | Months since the last commit to the project |
| 10% | Project age | Months since the first commit to the project |
| 10% | Commit frequency | Count of commits within the last year |

Some remarks on the choice of signals and parameters:

- By placing heavy weight on distinct contributors, we are prioritizing projects that have a large number of individual contributors, which is a strong indicator of community engagement and project health.
- We also weight distinct organizations, which helps to identify projects that have backing from multiple entities, indicating a broader support base.
- Source lines of code is a measure of project size, which can be an indicator of complexity and legacy of the codebase. Smaller projects that can be easily replaced should not be considered as critical.
- We intentionally avoid signals like issue activity as we find that they can be quite noisy, inconsistently measured across projects, and not necessarily indicative of criticality.

# Limitations

1. **VCS and DVCS platform specific (git and GitHub)** - Coming up with a set of signals that can be consistently measured across different platforms is a challenge. For example, using platform-specific signals like "number of forks" or "number of stars" is not possible for projects that are not hosted on GitHub. To accommodate this, we can use a set of signals that are more general and can be measured across different platforms. The signals we've chosen so far are based purely on the version control log of the open source project.
2. **Dependency information is not included** - A related issue of observability is that dependency information (which projects rely on others) can be hard to consistently gather. While SBOMs or packaging manifests do a decent job of recording runtime dependencies and other external artifacts a project relies upon, nuanced and potentially more important dependency relationships are imperfectly observed. For example, container technologies like Docker and Kubernetes rely heavily on the Linux kernel but neither directly "declare" the kernel as an import in packaging manifest.

# Future iterations

- Improved filtering (repository classification, NLP, etc.)
- Use a wider set of signals: move beyond VCS logs in some scalable way
- Extension: implement a user feedback loop
- Extension: incorporate dependency information and recursive scoring
- Extension: define alternative parameter sets for different use cases (e.g. security, widespread use, social impact, criticality for a specific user or organization, etc.)

# References

- [https://openssf.org/projects/criticality-score/](https://openssf.org/projects/criticality-score/)
- [https://github.com/ossf/criticality_score](https://github.com/ossf/criticality_score)
- [https://github.com/ossf/wg-securing-critical-projects](https://github.com/ossf/wg-securing-critical-projects)
- [https://openssf.org/blog/2023/07/28/understanding-and-applying-the-openssf-criticality-score-in-open-source-projects/](https://openssf.org/blog/2023/07/28/understanding-and-applying-the-openssf-criticality-score-in-open-source-projects/)
- [https://openssf.org/blog/2022/12/08/apples-and-apples-comparing-approaches-to-measuring-criticality-and-risk-at-the-openssf/](https://openssf.org/blog/2022/12/08/apples-and-apples-comparing-approaches-to-measuring-criticality-and-risk-at-the-openssf/)
- [https://todogroup.org/resources/guides/measuring-your-open-source-programs-success/\#what-to-track](https://todogroup.org/resources/guides/measuring-your-open-source-programs-success/#what-to-track)
- [https://chaoss.community/kbtopic/all-metrics-models/](https://chaoss.community/kbtopic/all-metrics-models/)