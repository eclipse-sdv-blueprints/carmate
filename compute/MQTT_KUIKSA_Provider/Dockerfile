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

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy script
COPY mqtt_kuksa_provider.py .
COPY config.json .

# Run script
CMD ["python", "mqtt_kuksa_provider.py"]
