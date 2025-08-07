Site deployed at: https://alvarorosado.es/

This is a [Hugo](https://gohugo.io/) site using [hugo-theme-stack](https://themes.gohugo.io/themes/hugo-theme-stack/).

To update Hugo, run:

```shell
brew upgrade hugo
```

To update the theme, run:

```shell
cd themes/hugo-theme-stack
# Get new tags from remote
git fetch --tags
# Get latest tag
latestTag=$(git describe --tags "$(git rev-list --tags --max-count=1)")
# Checkout latest tag
git checkout $latestTag
# Go back to the root of the project
cd ../..
# And commit the change to theme/hugo-theme-stack link
```
