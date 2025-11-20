# About

My personal blog where I share about technical and non-technical stuff that I came across. Powered by [Hugo](https://gohugo.io/) and [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

Found an error/typo? Submit an [issue](https://github.com/CookieHoodie/blog/issues) or create a [pull request](https://github.com/CookieHoodie/blog/pulls)!

# Dev
## Quick start
After cloning for the first time, we need to first fetch the PaperMod theme submodule.
```sh
$ git submodule update --init --recursive
```

Then, run the server.
```sh
$ docker compose up
```

## Upgrading PaperMod 
To update the theme to the latest commit, run
```sh
$ git submodule update --remote 
```