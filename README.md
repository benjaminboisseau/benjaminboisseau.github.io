# benjaminboisseau.github.io — personal site

Hand-written static site: plain HTML over a single stylesheet. No framework, no
generator, no JavaScript, no tracker. Bilingual (English / French).

```
index.html              home (EN)         fr/index.html        home (FR)
about.html              about (EN)        fr/about.html        about (FR)
posts/*.html            articles (EN)     fr/posts/*.html      articles (FR)
style.css               the one stylesheet
.nojekyll               tell GitHub Pages to serve files as-is
```

## Preview locally

Any static file server works, e.g. with BusyBox:

```
busybox httpd -f -p 8088
```

then open <http://localhost:8088/>.

## Publishing

Currently private. To publish via GitHub Pages, make the repo public (free tier)
and enable Pages on the `main` branch, root folder.
