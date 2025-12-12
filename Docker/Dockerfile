# Start from the gem5 deps image
FROM ghcr.io/gem5/ubuntu-24.04_all-dependencies:latest

# Build-time args to match host user
ARG USERNAME=dev
ARG UID=1000
ARG GID=1000

# Create a group/user with the same UID/GID as on the host
RUN groupadd -g ${GID} ${USERNAME} \
    && useradd -m -u ${UID} -g ${GID} -s /bin/bash ${USERNAME}

# Create the work directory and set ownership
RUN mkdir -p /workdir \
    && chown ${UID}:${GID} /workdir

# Update and install libs
RUN apt-get update && apt-get install -y libncurses-dev python3-venv python3-pybind11 pybind11-dev gdb && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/venv && /opt/venv/bin/pip install --no-cache-dir pandas scikit-learn pybind11 wheel setuptools pip
ENV PATH="/opt/venv/bin:${PATH}"

# Install Python deps
RUN pip3 install --no-cache-dir pandas

# Set working directory (this is where you'll mount your host dir)
WORKDIR /workdir

# Switch to the non-root user
USER ${USERNAME}

# Default shell
CMD ["/bin/bash"]
