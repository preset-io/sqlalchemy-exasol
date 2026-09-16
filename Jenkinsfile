#!/usr/bin/env groovy

// PR / branch validation for preset-io/sqlalchemy-exasol.
//
// Runs the offline unit suite (test/unit) only. test/integration drives a real
// Exasol database through exasol-integration-test-docker-environment and stays
// out of this pipeline.
//
// The dialect smoke check covers `exa.websocket`, the alias Superset connects
// through. The bare `exa` alias is deliberately NOT asserted here: upstream
// declares it as `sqlalchemy_exasol:base`, which resolves to a module rather
// than a Dialect class. Asserting it would fail on master today; it is tracked
// separately as part of the 7.1.3 entry-point fix.
//
// This pipeline intentionally publishes nothing: no sdist, no wheel, no
// registry upload, no git tag, no credentials.

properties([
    [$class: 'BuildDiscarderProperty',
     strategy: [$class: 'LogRotator', numToKeepStr: '20']],
])

podTemplate(
    imagePullSecrets: ['preset-pull'],
    nodeUsageMode: 'NORMAL',
    containers: [
        containerTemplate(
            alwaysPullImage: true,
            name: 'py-ci',
            image: 'preset/python:3.11.14-2026-05-22-ci',
            ttyEnabled: true,
            command: 'cat',
            resourceRequestCpu: '500m',
            resourceLimitCpu: '2000m',
            resourceRequestMemory: '1000Mi',
            resourceLimitMemory: '2000Mi',
        ),
    ]
) {
    node(POD_LABEL) {
        container('py-ci') {
            stage('Checkout') {
                checkout scm
            }

            stage('Install') {
                sh(script: 'python -m pip install --upgrade pip', label: 'Upgrade pip')
                sh(script: 'pip install -e .', label: 'Install sqlalchemy_exasol')
                sh(script: 'pip install pytest', label: 'Install test dependencies')
            }

            stage('Unit Tests') {
                try {
                    sh(
                        script: 'pytest test/unit --junitxml=junit-py.xml',
                        label: 'Unit tests',
                    )
                } finally {
                    junit allowEmptyResults: false, testResults: 'junit-py.xml'
                }
            }

            stage('Dialect Entry Points') {
                sh(
                    script: '''#!/usr/bin/env bash
set -eo pipefail
python - <<'PY'
from sqlalchemy.dialects import registry
from sqlalchemy.engine.default import DefaultDialect

EXPECTED = ["exa.websocket"]
failures = []
for name in EXPECTED:
    try:
        dialect = registry.load(name)
    except Exception as exc:
        failures.append(f"{name}: {type(exc).__name__}: {exc}")
        continue
    if not (isinstance(dialect, type) and issubclass(dialect, DefaultDialect)):
        failures.append(f"{name}: resolved to {dialect!r}, not a Dialect class")
        continue
    print(f"OK   {name} -> {dialect.__module__}.{dialect.__name__}")

if failures:
    for line in failures:
        print(f"FAIL {line}")
    raise SystemExit("sqlalchemy.dialects entry points are broken")
print("All asserted sqlalchemy.dialects entry points resolve to Dialect classes")
PY
''',
                    label: 'sqlalchemy.dialects entry points',
                )
            }
        }
    }
}
