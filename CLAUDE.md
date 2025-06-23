# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open-Meteo is a comprehensive open-source weather API built in Swift using the Vapor framework. It processes over 2TB of weather data daily from multiple national weather services and provides high-performance APIs with sub-10ms response times.

## Common Commands

### Build & Run
```bash
# Build and run in debug mode
swift build
swift run

# Build and run optimized release
swift build -c release
swift run -c release

# Run tests
swift test
swift test --filter ApiTests  # Run specific test class
```

### Development with Docker
```bash
# Development environment
docker build -f Dockerfile.development -t open-meteo-dev .
docker run -it --security-opt seccomp=unconfined -p 8080:8080 \
  -v ${PWD}:/app -v open-meteo-data:/app/data open-meteo-dev

# Docker compose for full stack
docker compose up
```

### Data Operations
```bash
# Sync weather data
openmeteo-api sync ecmwf_ifs025 temperature_2m
openmeteo-api sync dwd_icon,gfs_global temperature_2m --past-days 7

# Download weather models
openmeteo-api download-ecmwf --run 00
openmeteo-api download-gfs --run 06

# Utility commands
openmeteo-api migration
openmeteo-api benchmark
openmeteo-api download-dem
```

## Architecture

### Core Components
- **HTTP API Server**: Vapor-based REST API serving weather data
- **Custom File Database**: Optimized binary format for time-series weather data in `./data`
- **Download Commands**: Weather model downloaders from national services  
- **Sync System**: Distributed data synchronization across nodes

### Key Directories
- `Sources/App/Controllers/` - API route controllers for different weather services
- `Sources/App/Helper/` - Utilities including download, file I/O, and meteorological calculations
- `Sources/App/Domains/` - Weather domain definitions and grid projections
- `Sources/App/Commands/` - CLI command implementations for data operations
- `Sources/App/*/` - Weather service integrations (ECMWF, GFS, ICON, etc.)

### Main API Routes
- `/v1/forecast` - Primary weather forecast API
- `/v1/archive` - Historical weather data (ERA5)
- `/v1/dwd-icon`, `/v1/gfs`, `/v1/meteofrance`, `/v1/ecmwf` - Service-specific APIs
- `/v1/air-quality`, `/v1/marine`, `/v1/ensemble` - Specialized forecasts

## Development Environment

### Prerequisites
- Swift 6.0+, Vapor 4.89.0+
- Modern CPU with SIMD/AVX2 support
- 16GB+ RAM recommended
- NVMe SSD for high IOPS performance
- 150GB+ disk space for full datasets

### Environment Variables
```bash
ENABLE_PARQUET=TRUE          # Enable Parquet format support
DATA_DIRECTORY=/path/to/data # Custom data directory
REMOTE_DATA_DIRECTORY=https://openmeteo.s3.amazonaws.com/data/
CACHE_SIZE=10GB              # Memory cache configuration
```

### Performance Features
- Cross-module optimization enabled
- Architecture-specific compiler optimizations
- Memory-mapped file operations for weather data
- Custom time-series database format (OM-File-Format)
- Response compression and high connection backlog

## Testing

The test suite covers API functionality, meteorological calculations, time handling, and data format operations. Tests are located in `Tests/AppTests/`.