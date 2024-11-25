# ChatDB GPU - Docker Setup Guide

This repository contains a Docker-based setup for running a Streamlit application with Ollama, optimized for GPU usage, supporting both .GGUF and Ollama models.

## Prerequisites

### System Requirements
- NVIDIA GPU
- Docker Desktop
- WSL2 (for Windows users)
- NVIDIA GPU drivers
- NVIDIA Container Toolkit

### Windows-Specific Setup
1. Install and Configure WSL2
   ```bash
   wsl --install
   wsl --set-default-version 2
   ```

2. Docker Desktop Settings
   - Enable "Use WSL 2 based engine" in Settings > General
   - Enable WSL Integration in Settings > Resources > WSL Integration

3. Verify GPU Support
   ```bash
   docker run --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
   ```

## Installation

1. Pull Docker Images
   ```bash
   docker pull parthi97/llm-sql-gpu:latest
   docker pull parthi97/ollama:latest
   ```

2. Create `docker-compose.yml`
   ```yaml
   version: '3.8'
   services:
     streamlit:
       image: parthi97/llm-sql-gpu:latest
       ports:
         - "8501:8501"
       deploy:
         resources:
           reservations:
             devices:
               - driver: nvidia
                 count: 1
                 capabilities: [gpu]
       networks:
         - llm_network
       environment:
         - OLLAMA_HOST=http://host.docker.internal:11434
         - NVIDIA_VISIBLE_DEVICES=all
         - NVIDIA_DRIVER_CAPABILITIES=compute,utility
       volumes:
         - model_data:/app/models
       depends_on:
         - ollama

     ollama:
       image: parthi97/ollama:latest
       ports:
         - "11434:11434"
       environment:
         - OLLAMA_HOST=0.0.0.0
         - OLLAMA_MODELS_MEMORY_USAGE=60
         - CUDA_VISIBLE_DEVICES=0
         - NVIDIA_VISIBLE_DEVICES=all
         - NVIDIA_DRIVER_CAPABILITIES=compute,utility
         - CUDA_LAUNCH_BLOCKING=1
         - PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:512
       deploy:
         resources:
           reservations:
             devices:
               - driver: nvidia
                 count: 1
                 capabilities: [gpu]
               options:
                 memory: 3G
       volumes:
         - model_data:/root/.ollama
       networks:
         - llm_network

   networks:
     llm_network:
       driver: bridge

   volumes:
     model_data:
   ```

## Running the Application

### Method : Using Command Line (From the current working directory)
```bash
docker-compose up
```

## Model Setup and Usage

### Using .GGUF Models
1. For using local .GGUF models, you need to mount the model file when running the container:
   ```bash
   docker run --gpus all -v <fully qualified path to .gguf>:/app/models/model.gguf -p 8501:8501 <REPOSITORY>
   ```

   Example:
   ```bash
   docker run --gpus all -v G:\chatdb_llm\natural-sql-7b.Q4_K_M.gguf:/app/models/model.gguf -p 8501:8501 parthi97/llm-sql-gpu:latest
   ```

2. In the Streamlit UI:
   - Under "Model Path" input field, enter: `/app/models/model.gguf`
   - This path maps to your locally mounted .GGUF file

### Using Ollama Models
1. First, identify the Ollama container ID:
   ```bash
   docker ps
   ```
   Look for the container running the `parthi97/ollama:latest` image

2. Pull the desired model using the container ID:
   ```bash
   docker exec -it <container_id> ollama pull <model_name>
   ```

   Example:
   ```bash
   docker exec -it 8ff644f8a2c7 ollama pull llama3.1:8b
   ```

3. In the Streamlit UI:
   - Select "Ollama" as your model type
   - Choose the pulled model from the available options

### Model Usage Best Practices
1. For .GGUF Models:
   - Ensure the model file is in a location accessible to Docker
   - Use absolute paths when mounting volumes
   - Verify file permissions are correct

2. For Ollama Models:
   - Pull models after the container is running
   - Monitor GPU memory usage while pulling models
   - Consider available GPU memory when selecting models

3. General Recommendations:
   - Start with smaller models if GPU memory is limited
   - Monitor system resources during model loading
   - Ensure proper model path configuration in Streamlit UI

## Accessing the Application
- Streamlit Interface: http://localhost:8501
- Ollama API: http://localhost:11434

## Verification and Troubleshooting

### Check Container Status
```bash
docker ps
docker-compose logs streamlit
docker-compose logs ollama
```

### Verify GPU Access
```bash
nvidia-smi
docker run --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

### Common Issues and Solutions
1. GPU not detected:
   - Verify NVIDIA drivers are installed
   - Check Docker Desktop GPU settings
   - Ensure WSL2 GPU support is enabled

2. Container fails to start:
   - Check docker-compose logs
   - Verify all ports are available
   - Ensure sufficient GPU memory

3. Model-Related Issues:
   - Model Not Found:
     - Verify the correct path is mounted for .GGUF models
     - Check if Ollama models were pulled successfully
     - Ensure proper permissions on model files
   
   - GPU Memory Issues:
     - Monitor GPU memory usage with `nvidia-smi`
     - Consider using quantized models for better memory efficiency
     - Close other GPU-intensive applications
   
   - Model Loading Errors:
     - Check container logs for specific error messages
     - Verify model compatibility with the system
     - Ensure sufficient disk space for model files

## Stopping the Application
```bash
docker-compose down        # Stop containers
docker-compose down -v     # Stop containers and remove volumes
```

## Resource Requirements
- Minimum 3GB GPU Memory
- Approximately 10GB Disk Space for images
- NVIDIA GPU with CUDA support
