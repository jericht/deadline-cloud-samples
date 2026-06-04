# Maya Docker image for AWS Deadline Cloud container rendering
# Base: ASWF ci-base (Rocky Linux 8, VFX Platform compliant, CUDA included)
#
# Maya is NOT freely downloadable. You must provide the Maya installer tarball.
# Download from: https://www.autodesk.com/products/maya/overview
# Place the .tgz file in the `installer/` directory next to this Dockerfile.

ARG VFX_PLATFORM_YEAR=2026
FROM aswf/ci-base:${VFX_PLATFORM_YEAR}

ARG MAYA_VERSION=2025
ARG ADAPTOR_VERSION=0.15.15

ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES=compute,utility,graphics

# Maya environment variables
ENV MAYA_LOCATION=/usr/autodesk/maya${MAYA_VERSION}
ENV PATH=${MAYA_LOCATION}/bin:$PATH
ENV MAYA_DISABLE_CIP=1
ENV MAYA_DISABLE_CER=1
ENV MAYA_DISABLE_ADP=1
ENV MAYA_NO_STANDALONE_ATEXIT=1

# --- System dependencies ---
# ASWF base has most deps; Maya needs a few extras
RUN yum install -y \
    libXxf86vm libXi libXrender libSM libICE libXext libXfixes \
    libXcomposite libXdamage libXrandr libXinerama libXt libXtst \
    libXmu libXpm libxkbfile alsa-lib libtiff nss pcre \
    libjpeg-turbo fontconfig freetype mesa-libEGL mesa-libGLU \
    xorg-x11-fonts-100dpi xorg-x11-fonts-75dpi dbus \
    && yum clean all

# --- Install Maya ---
# Extract RPM directly with rpm2cpio+cpio (much faster than alien).
COPY installer/ /tmp/maya_installer/
RUN cd /tmp/maya_installer \
    && tar -xzf Autodesk_Maya_*_Linux_64bit.tgz \
    && for rpm in Packages/Maya*.rpm; do \
        echo "Extracting: $(basename $rpm)" \
        && rpm2cpio "$rpm" | cpio -idm 2>/dev/null; \
    done \
    && cp -a /tmp/maya_installer/usr/autodesk /usr/autodesk \
    && (cp -a /tmp/maya_installer/usr/bin/* /usr/bin/ 2>/dev/null || true) \
    && (cp -a /tmp/maya_installer/usr/lib/* /usr/lib/ 2>/dev/null || true) \
    && cd / \
    && rm -rf /tmp/maya_installer \
    && ldconfig \
    && ln -sf /usr/autodesk/maya${MAYA_VERSION}/bin/maya${MAYA_VERSION} /usr/autodesk/maya${MAYA_VERSION}/bin/maya \
    && maya -batch -v

# --- Install the Deadline Cloud Maya adaptor ---
RUN python3 -m pip install --no-cache-dir deadline-cloud-for-maya==${ADAPTOR_VERSION}

# --- Install user-supplied plugins (optional) ---
# Arnold (MtoA) is typically bundled with Maya. Additional renderers
# (V-Ray, Redshift) can be placed in plugins/ as .run installers.
ENV MAYA_MODULE_PATH=/opt/maya-plugins
RUN mkdir -p /opt/maya-plugins && chmod -R 777 /opt/maya-plugins
COPY plugins/ /tmp/plugins/
RUN for installer in /tmp/plugins/*.run; do \
        [ -f "$installer" ] || continue; \
        chmod +x "$installer" && "$installer" --noexec --target /tmp/plugin_extract \
        && echo "Installed plugin: $(basename $installer)"; \
    done \
    && rm -rf /tmp/plugins /tmp/plugin_extract

# --- Verify installations ---
RUN maya -batch -v && maya-openjd --help > /dev/null 2>&1

CMD ["bash", "-c", "sleep infinity"]
