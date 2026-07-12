ARG BUILD_FROM
# The MCP server no longer compiles firmware itself — it delegates every
# build/flash/validate/logs call to the ESPHome Device Builder dashboard
# running in the official ESPHome add-on. So there's no need for the heavy
# ESPHome (Debian/glibc + ESP toolchains + PlatformIO) base image or its
# pinned esphome version; a slim Python base (set in build.yaml) is enough.
FROM ${BUILD_FROM}

COPY requirements.txt /tmp/
RUN pip3 install --no-cache-dir -r /tmp/requirements.txt \
    && rm /tmp/requirements.txt

# Copy server code
COPY server/ /opt/server/
COPY run.sh /opt/

RUN chmod a+x /opt/run.sh

WORKDIR /opt

CMD ["/opt/run.sh"]
