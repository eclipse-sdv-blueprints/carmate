# ********************************************************************************
# Copyright (c) 2026 Ferdinand Steinbeis Institut AIoT lab[ and others] 
#
# This program and the accompanying materials are made available under the 
# terms of the Eclipse Public License 2.0 which is available at
# https://www.eclipse.org/legal/epl-2.0.
#
# SPDX-License-Identifier: EPL-2.0 
# ********************************************************************************

FROM python:3.11-slim

WORKDIR /app

# Install git (needed to clone repo)
RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Clone repo
RUN git clone https://github.com/eclipse-uprotocol/up-transport-zenoh-python.git \
    && cd up-transport-zenoh-python \
    && git checkout c2bda41

# Set working directory inside repo
WORKDIR /app/up-transport-zenoh-python

# Install repo dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy your script into container
COPY . /app/up-transport-zenoh-python/

# Run script from repo context
CMD ["python", "carmate_agents.py"]