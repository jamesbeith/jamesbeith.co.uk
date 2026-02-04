---
title: "How to catch missing referenced files in Django static files manifest"
date: "2025-02-27T12:00:00+11:00"
---

I recently had a deployment fail with the following [WhiteNoise](https://whitenoise.readthedocs.io/en/stable/) error.

```text
-----> $ python src/manage.py collectstatic --noinput
       Post-processing 'polls/style.css' failed!
       whitenoise.storage.MissingFileError: The file 'polls/style.css.map' could not be found with <whitenoise.storage.CompressedManifestStaticFilesStorage object at 0x7f5c4b1970e0>.
       The CSS file 'polls/style.css' references a file which could not be found:
         polls/style.css.map
       Please check the URL references in this CSS file, particularly any
       relative paths which might be pointing to the wrong location.
```

It was easy enough to fix, by adding the missing `style.css.map` file, but I wanted to write a test that would help prevent this from happening again in the future. Here’s what I started with.

```python
import tempfile

import pytest
from pytest_django.fixtures import SettingsWrapper
from whitenoise import storage


def test_collect_static_files_manifest(settings: SettingsWrapper) -> None:
    """
    Test that all referenced static files exist in the manifest.
    """
    with tempfile.TemporaryDirectory() as directory:
        settings.STATIC_ROOT = directory
        try:
            call_command("collectstatic", no_input=True)
        except storage.MissingFileError:
            pytest.fail("Missing file in the static files manifest")
```

However, when I ran the test with the `polls/style.css.map` file missing, it didn’t fail. I checked the configuration of WhiteNoise in my Django settings file.

```python
from django.conf import STATICFILES_STORAGE_ALIAS

from environs import env

MIDDLEWARE = [
    ...,
    "whitenoise.middleware.WhiteNoiseMiddleware",
    ...,
]

STORAGES = {
    STATICFILES_STORAGE_ALIAS: {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}

if env.bool("CI", False):
    MIDDLEWARE.remove("whitenoise.middleware.WhiteNoiseMiddleware")
    STORAGES[STATICFILES_STORAGE_ALIAS] = {
        "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
    }
```

Turned out when running the tests I had static files handled by Django, without manifest support, instead of WhiteNoise. I remembered that, as [mentioned in the Django docs](https://docs.djangoproject.com/en/5.1/ref/contrib/staticfiles/#django.contrib.staticfiles.storage.ManifestStaticFilesStorage.manifest_strict), you shouldn’t use a backend with manifest support when running tests.

<!-- vale off -->

> Due to the requirement of running `collectstatic`, this storage typically shouldn’t be used when running tests as `collectstatic` isn’t run as part of the normal test setup. During testing, ensure that `staticfiles` storage backend in the `STORAGES` setting is set to something else like `'django.contrib.staticfiles.storage.StaticFilesStorage'` (the default).

<!-- vale on -->

If you do then you’d get errors similar to.

```text
.venv/lib/python3.13/site-packages/django/contrib/staticfiles/storage.py:516: in stored_name
    raise ValueError(
E   ValueError: Missing staticfiles manifest entry for 'polls/style.css'
```

I wanted to consistently use WhiteNoise and remove the conditional logic within my Django settings file. That is, I didn’t want settings defined based on knowing what environment the application was running in. I made the following change.

```diff
  STORAGES = {
      STATICFILES_STORAGE_ALIAS: {
-         "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
+         "BACKEND": env.str("WHITENOISE_BACKEND", default="whitenoise.storage.CompressedManifestStaticFilesStorage"),
      },
  }

- if env.bool("CI", False):
-     MIDDLEWARE.remove("whitenoise.middleware.WhiteNoiseMiddleware")
-     STORAGES[STATICFILES_STORAGE_ALIAS] = {
-         "BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage",
-     }
```

In my test environment I set the environment variable to the WhiteNoise backend without manifest support.

```text
WHITENOISE_BACKEND="whitenoise.storage.StaticFilesStorage"
```

I then updated the new test to explicitly use the WhiteNoise backend with manifest support.

```diff
  import tempfile

+ from django.conf import STATICFILES_STORAGE_ALIAS
+
  import pytest
  from pytest_django.fixtures import SettingsWrapper
  from whitenoise import storage


  def test_collect_static_files_manifest(settings: SettingsWrapper) -> None:
      """
      Test that all referenced static files exist in the manifest.
      """
+     settings.STORAGES = {
+         STATICFILES_STORAGE_ALIAS: {
+             "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
+         },
+     }
      with tempfile.TemporaryDirectory() as directory:
          settings.STATIC_ROOT = directory
          try:
              call_command("collectstatic", no_input=True)
          except storage.MissingFileError:
              pytest.fail("Missing file in the static files manifest")
```

Now the test failed as expected with the `polls/style.css.map` file missing, and passed when I added it.

Bear in mind that I’m using the WhiteNoise backend with compression support in this test, to mimic my production environment. That comes with the overhead of also compressing all files during `collectstatic`. To avoid that, whilst preserving the same behaviour of the test, consider switching to Django’s `"django.contrib.staticfiles.storage.ManifestStaticFilesStorage"` backend and catching `ValueError` instead of `storage.MissingFileError`.

One last thing I noticed was that I was now getting the following warning when my other tests ran.

```text
UserWarning: No directory at: /.../static_root/
```

This was because my test environment runs with Django’s `DEBUG = False`. The simplest way to fix this was to enable the [`WHITENOISE_AUTOREFRESH`](https://whitenoise.readthedocs.io/en/stable/django.html#whitenoise-makes-my-tests-run-slow) setting in my test (and local) environment. This stops WhiteNoise from scanning static files on start up but other than that its behaviour should be exactly the same.

```diff
  STORAGES = {
      STATICFILES_STORAGE_ALIAS: {
          "BACKEND": env.str("WHITENOISE_BACKEND", default="whitenoise.storage.CompressedManifestStaticFilesStorage"),
      },
  }

+ # For both performance and security reasons, `WHITENOISE_AUTOREFRESH` should not be used in production.
+ WHITENOISE_AUTOREFRESH = env.bool("WHITENOISE_AUTOREFRESH", False)
```

In my test and local environments I set the environment variable.

```text
WHITENOISE_AUTOREFRESH=true
```
