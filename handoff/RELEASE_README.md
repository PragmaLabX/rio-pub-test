# How to cut a release

```bash
git checkout main && git pull
npm version patch   # patch | minor | major
git push --follow-tags
```

That's it — pushing the tag triggers the workflow, which runs the tests,
checks the tag against `package.json`'s version, and publishes the package
to npm automatically. Check the **Actions** tab if it fails.
