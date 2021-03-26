# keylime-docs

[![Documentation Status](https://readthedocs.org/projects/keylime-docs/badge/?version=latest)](https://keylime-docs.readthedocs.io/en/latest/?badge=latest)

All Keylime documentation is formatted in [reStructuredText](https://en.wikipedia.org/wiki/ReStructuredText)

The documentation project is built using [sphinx](http://www.sphinx-doc.org)

Documentation is hosted on [readthedocs](https://readthedocs.org/)

## Building the Documentation


```
pip install -r requirements.txt`
cd docs
make html
```

This will report any stuff like formatting errors.

You then go into `_build/html` and the rendered files will be there

```
 ls _build/html                                                                                                                                                                      
404.html  developers.html  genindex.html  http-routingtable.html  _images  index.html  installation.html  objects.inv  rest_apis.html  search.html  searchindex.js  security.html  _sources  _static  user_guide  user_guide.html
```

Also when you make the pull request, readthedocs will do the same as the above and report on build status.