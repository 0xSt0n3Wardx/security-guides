FROM wazuh/wazuh-manager:4.11.1
USER root
RUN apt-get update && apt-get install -y python3 python3-pip
RUN python3 -m pip install --no-cache-dir --target=/usr/lib/python3.9/site-packages requests
USER wazuh