FROM python:3.13.15-slim-bookworm

# procps: agentsweep's preflight check shells out to `ps`. git: lets the offline agent inspect history.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git procps \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir uv==0.12.23

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Install the project plus the dev and fast (RE2) extras exactly as pinned in uv.lock.
# Everything the test suite needs is fetched here; the audit runs offline afterwards.
ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    UV_FROZEN=1
RUN uv sync --frozen --extra dev --extra fast --python /usr/local/bin/python3
ENV PATH=/src/.venv/bin:$PATH \
    UV_OFFLINE=1 \
    AGENTSWEEP_NO_UPDATE=1

# Smoke-check the CLI, then run the tests without letting a failure abort the build.
RUN agentsweep --version
RUN python -m pytest tests/ -q || echo "WARNING: agentsweep tests reported failures"
