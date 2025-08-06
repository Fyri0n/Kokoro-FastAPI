FROM --platform=$BUILDPLATFORM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# Install Python and other dependencies
RUN apt-get update -y &&  \
    apt-get install -y python3.10 python3-venv espeak-ng espeak-ng-data git libsndfile1 curl ffmpeg g++ &&  \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    mkdir -p /usr/share/espeak-ng-data &&  \
    ln -s /usr/lib/*/espeak-ng-data/* /usr/share/espeak-ng-data/ && \
    curl -LsSf https://astral.sh/uv/install.sh | sh && \
    mv /root/.local/bin/uv /usr/local/bin/ && \
    mv /root/.local/bin/uvx /usr/local/bin/  && \
    useradd -m -u 1001 appuser && \
    mkdir -p /app/api/src/models/v1_0 && \
    chown -R appuser:appuser /app && \
    chmod -R 755 /app
    
USER appuser
WORKDIR /app

# Copy dependency files
COPY --chown=appuser:appuser pyproject.toml ./pyproject.toml

# Install dependencies with GPU extras
RUN uv venv --python 3.10 && \
    uv sync --extra gpu --no-cache && \
    /app/.venv/bin/python -m pip install --force-reinstall torch==1.13.1+cu118 torchvision==0.14.1+cu118 torchaudio==0.13.1 \
    -f https://download.pytorch.org/whl/torch_stable.html

# Copy project files including models
COPY --chown=appuser:appuser api ./api
COPY --chown=appuser:appuser web ./web
COPY --chown=appuser:appuser docker/scripts/ ./
RUN chmod +x ./entrypoint.sh && \
    mkdir -p /app/api/temp_files && \
    chmod -R 777 /app/api/temp_files

# Set all environment variables in one go
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    UV_LINK_MODE=copy \
    USE_GPU=true \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data \
    DEVICE="gpu"
    
# Set model environment variables
ENV MODEL_DIR=src/models

# Download model during build time
RUN python download_model.py --output /app/api/src/models/v1_0 && \
    # Ensure proper permissions on all model directories
    chmod -R 777 /app/api/src/models && \
    # Create symbolic link to ensure v1_0 files are found in the right place
    ln -sf /app/api/src/models/v1_0/kokoro-v1_0.pth /app/api/src/models/v1_0.pth && \
    ln -sf /app/api/src/models/v1_0/config.json /app/api/src/models/v1_0.json

# Run FastAPI server through entrypoint.sh
CMD ["./entrypoint.sh"]
