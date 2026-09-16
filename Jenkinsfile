// Preset publication pipeline for the internal fallback build of this dialect.
//
// Scope and safety properties:
//
//   * Stable artifacts are published ONLY from reviewed, merged Preset history
//     (the `master` branch). Pull-request builds build and verify, then stop
//     before any upload.
//   * The upload is atomic and never overwrites: S3 `put-object` is issued with
//     `--if-none-match '*'`, so a key that already exists fails the build rather
//     than being replaced. A pre-flight `head-object` gives the same answer with
//     a clearer message, but the conditional write is what actually guarantees it.
//   * Credentials are bound at runtime from the existing `ci-user` Jenkins
//     credential. Nothing is stored in this repository, printed, or written to
//     the build log.
//   * Object storage is addressed through the S3 API only. No index endpoint is
//     referenced here.
//   * The published artifact is the source distribution, matching how Preset
//     already consumes rebuilt third-party Python packages.
//   * The build is verified reproducible: the distribution is built twice and
//     the two SHA-256 digests must match before anything is uploaded.
//   * This pipeline does not tag, release, or push to the repository. Tagging
//     was deliberately left out so the job needs no repository write credential.

LIB_NAME = 'sqlalchemy-exasol'
BUCKET = 'preset-pypi'
POETRY_VERSION = '2.3.0'

String version = ""
String sdist = ""
String digest = ""
String key = ""

podTemplate(
    imagePullSecrets: ['preset-pull'],
    nodeUsageMode: 'NORMAL',
    containers: [
        containerTemplate(
            alwaysPullImage: true,
            name: 'ci',
            image: 'preset/ci:latest',
            ttyEnabled: true,
            command: 'cat',
            resourceRequestCpu: '100m',
            resourceLimitCpu: '200m',
            resourceRequestMemory: '1000Mi',
            resourceLimitMemory: '2000Mi',
        ),
        containerTemplate(
            alwaysPullImage: true,
            name: 'py-ci',
            image: 'preset/python:3.10.13-2024-02-21-ci',
            ttyEnabled: true,
            command: 'cat'
        )
    ]
) {
    node(POD_LABEL) {
        container('py-ci') {
            stage('Checkout') {
                checkout scm
                sh(script: "git config --global --add safe.directory '*'", label: 'Trust workspace')
            }

            stage('Install Build Tooling') {
                sh(script: "pip install --no-cache-dir 'poetry==${POETRY_VERSION}'", label: 'Install poetry')
            }

            stage('Unit Tests') {
                sh(script: 'poetry install --no-interaction', label: 'Install project')
                sh(script: 'poetry run -- python -m pytest test/unit -q', label: 'Run unit tests')
            }

            stage('Build and Verify Reproducibility') {
                version = sh(
                    script: 'poetry version --short',
                    returnStdout: true,
                    label: 'Read project version'
                ).trim()
                sdist = "sqlalchemy_exasol-${version}.tar.gz"
                key = "${LIB_NAME}/${sdist}"

                sh(script: 'rm -rf dist build-a build-b upload', label: 'Clean build outputs')
                sh(script: 'poetry build --format sdist && mv dist build-a', label: 'First build')
                sh(script: 'poetry build --format sdist && mv dist build-b', label: 'Second build')

                // A rebuild from the same commit must produce byte-identical
                // bytes, otherwise the published digest cannot be re-derived.
                sh(
                    script: """
                        set -eu
                        a=\$(sha256sum build-a/${sdist} | cut -d' ' -f1)
                        b=\$(sha256sum build-b/${sdist} | cut -d' ' -f1)
                        echo "build A sha256: \$a"
                        echo "build B sha256: \$b"
                        if [ "\$a" != "\$b" ]; then
                            echo "Build is not reproducible; refusing to publish."
                            exit 1
                        fi
                    """,
                    label: 'Compare digests'
                )

                digest = sh(
                    script: "sha256sum build-a/${sdist} | cut -d' ' -f1",
                    returnStdout: true,
                    label: 'Record digest'
                ).trim()

                sh(script: "mkdir -p upload && cp build-a/${sdist} upload/", label: 'Stage artifact')
                archiveArtifacts artifacts: "upload/${sdist}", fingerprint: true
                echo "Artifact ${sdist} sha256 ${digest}"
            }

            stage('Verify Installable Artifact') {
                // Install the exact artifact that would be published and assert
                // the packaged metadata and dialect entry points are intact.
                sh(
                    script: """
                        set -eu
                        python -m venv /tmp/verify
                        /tmp/verify/bin/pip install --quiet upload/${sdist}
                        /tmp/verify/bin/python - <<'EOF'
import importlib.metadata as md
import sqlalchemy as sa

dist = md.distribution("sqlalchemy_exasol")
assert dist.version == "${version}", dist.version
groups = {(e.group, e.name) for e in dist.entry_points}
for name in ("exa", "exa.websocket"):
    assert ("sqlalchemy.dialects", name) in groups, name
for url in ("exa://u:p@h:8563/S", "exa+websocket://u:p@h:8563/S"):
    dialect = type(sa.create_engine(url).dialect)
    assert dialect.__name__ == "EXADialect_websocket", dialect
    assert "is_disconnect" in dialect.__dict__, "disconnect fix missing"
print("verified", dist.version)
EOF
                    """,
                    label: 'Install and inspect artifact'
                )
            }
        }

        container('ci') {
            stage('Publish') {
                if (env.BRANCH_NAME != 'master') {
                    echo "Branch ${env.BRANCH_NAME} is not master; built and verified only, nothing published."
                    return
                }

                withCredentials([
                    [
                        $class           : 'AmazonWebServicesCredentialsBinding',
                        credentialsId    : 'ci-user',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY',
                    ]
                ]) {
                    sh(
                        script: """
                            set -eu
                            if aws s3api head-object --bucket '${BUCKET}' --key '${key}' >/dev/null 2>&1; then
                                echo "${key} already exists. Published artifacts are immutable; bump the version."
                                exit 1
                            fi
                        """,
                        label: 'Refuse to republish an existing version'
                    )

                    // Jenkins currently ships AWS CLI v1, whose s3api command
                    // cannot express If-None-Match. Use the current boto3 API so
                    // the write remains atomic rather than weakening the guard.
                    sh(
                        script: """
                            set -eu
                            python -m pip install --quiet 'boto3>=1.36,<2'
                            BUCKET='${BUCKET}' KEY='${key}' ARTIFACT='upload/${sdist}' \
                                python -c 'import os, boto3; artifact = open(os.environ["ARTIFACT"], "rb"); boto3.client("s3").put_object(Bucket=os.environ["BUCKET"], Key=os.environ["KEY"], Body=artifact, IfNoneMatch="*")'
                        """,
                        label: 'Atomic upload'
                    )

                    sh(
                        script: """
                            set -eu
                            aws s3api head-object --bucket '${BUCKET}' --key '${key}' >/dev/null
                            echo "Published ${key}"
                            echo "sha256 ${digest}"
                        """,
                        label: 'Confirm published object'
                    )
                }
            }
        }
    }
}
