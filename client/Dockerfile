FROM python:3.14-slim-bookworm
RUN apt-get update \
    && apt-get install -y --no-install-recommends openssh-server \
    && rm -rf /var/lib/apt/lists/* /etc/ssh/ssh_host_* \
    && useradd --create-home --shell /bin/sh --password x tunnel \
    && mkdir -p /run/sshd /etc/ssh/authorized_keys \
    && pip install --no-cache-dir wg-bridge==1.0.3

# Docker forwards UDP to the container interface, not its loopback.
RUN python - <<'PY'
from importlib.metadata import distribution

path = distribution("wg-bridge").locate_file("wg_bridge/client.py")
source = path.read_text()
old = 'udp_sock.bind(("127.0.0.1", wg_port))'
new = 'udp_sock.bind(("0.0.0.0", wg_port))'
if source.count(old) != 1:
    raise SystemExit("Unexpected wg-bridge source: UDP bind patch not applied")
patched = source.replace(old, new, 1)
compile(patched, str(path), "exec")
path.write_text(patched)
PY
