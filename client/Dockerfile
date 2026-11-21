FROM python:3.14-slim-bookworm
RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-server \
    && rm -rf /var/lib/apt/lists/* /etc/ssh/ssh_host_* \
    && useradd --create-home --shell /bin/sh --password x tunnel \
    && mkdir -p /run/sshd /etc/ssh/authorized_keys \
    && pip install --no-cache-dir wg-bridge==1.0.3
