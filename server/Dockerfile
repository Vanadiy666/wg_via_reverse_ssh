FROM python:3.14-slim-bookworm
RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-client \
    && rm -rf /var/lib/apt/lists/* \
    && pip install --no-cache-dir wg-bridge==1.0.3
