# AstroPaper posts

This branch contains the posts and GitHub Pages deployment workflow for the
AstroPaper site.

## Blog language

Set the `BLOG_LANG` GitHub Repository variable to the language configured for
the AstroPaper site (for example, `en` or `ja`). The deployment workflow passes
this value to the site build. Japanese builds also download and cache the Noto
Sans JP fonts used to render dynamic Open Graph images; other languages keep
using the font handling provided by the AstroPaper site.

## Optional giscus comments

Comments are disabled unless all of the following GitHub Repository variables
are configured:

- `GISCUS_REPO`
- `GISCUS_REPO_ID`
- `GISCUS_CATEGORY`
- `GISCUS_CATEGORY_ID`

Add them under **GitHub repository → Settings → Secrets and variables →
Actions → Variables**. Do not add `PUBLIC_` to the Repository variable names;
the deployment workflow maps them to Astro's public build-time environment
variables.

The repository selected in `GISCUS_REPO` must also have
[GitHub Discussions enabled](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/enabling-or-disabling-github-discussions-for-a-repository),
and the [giscus GitHub App](https://github.com/apps/giscus) must be installed and
configured for that repository. Use the [giscus configuration
page](https://giscus.app/) to obtain the repository and category IDs.
