# Unreleased

## Summary

## Bug fixes

* Preserve fractional seconds and naive wall time in websocket TIMESTAMP results.
* Recognize wrapped PyExasol communication errors as disconnects, enabling
  SQLAlchemy's pool pre-ping to replace stale connections on the first checkout.
  Server query/authentication errors are not disconnects; in-flight SQL is not replayed.

See issue #807.

## Preset internal build

`7.1.3.1` is a Preset-owned rebuild of upstream `7.1.3` carrying only the two bug
fixes above. It exists solely as a release-blocker fallback while
[exasol/sqlalchemy-exasol#808](https://github.com/exasol/sqlalchemy-exasol/pull/808)
is under maintainer review, and it is published only to Preset's internal index.
The four-component version follows Preset's existing convention for rebuilt
third-party packages (for example `pyhive-0.7.0.1`); upstream has only ever
released three-component versions, and `7.1.3.1` sorts above `7.1.3` and below
`7.1.4`, so an upstream release supersedes it automatically. It is never
published to public PyPI.
