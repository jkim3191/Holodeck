# AI2Holodeck Codebase Analysis for Claude

## Project Overview
AI2Holodeck is a language-guided 3D embodied AI environment generation system built on AI2-THOR. It generates procedural indoor scenes from natural language descriptions using LLMs (primarily GPT-4o-2024-05-13), CLIP models for visual understanding, and constraint solvers for object placement.

## Core Architecture

### Main Entry Point
- **File**: `/home/ycho358/GitHub/Holodeck-Jae/ai2holodeck/main.py`
- **Purpose**: CLI interface supporting three modes:
  - `generate_single_scene`: Generate one scene from a query
  - `generate_multi_scenes`: Batch generate from query file  
  - `generate_variants`: Create variations of existing scenes

### Scene Generation Pipeline
The system follows a hierarchical generation approach in `/home/ycho358/GitHub/Holodeck-Jae/ai2holodeck/generation/holodeck.py`:

1. **Floor Plan Generation** (`generate_rooms`) - Creates room layouts with materials
2. **Wall Generation** (`generate_walls`) - Defines wall structure and heights  
3. **Door Generation** (`generate_doors`) - Places doors between rooms and exterior
4. **Window Generation** (`generate_windows`) - Adds windows to walls
5. **Object Selection** (`select_objects`) - LLM-driven object selection for each room
6. **Floor Object Placement** - Constraint-based placement using DFS or MILP solvers
7. **Wall Object Placement** - Paintings, shelves, switches, etc.
8. **Small Object Generation** - Decorative items placed on receptacles
9. **Ceiling Objects** (optional) - Lights and fans
10. **Lighting & Materials** - Automatic lighting and material assignment

### Key Components

#### Core Generators (`/home/ycho358/GitHub/Holodeck-Jae/ai2holodeck/generation/`)
- **`rooms.py`**: `FloorPlanGenerator` - Rectangular room layouts with materials
- **`object_selector.py`**: `ObjectSelector` - LLM-driven object selection from Objaverse
- **`floor_objects.py`**: `FloorObjectGenerator` - Constraint-based furniture placement
- **`wall_objects.py`**: `WallObjectGenerator` - Wall-mounted object placement
- **`small_objects.py`**: `SmallObjectGenerator` - Decorative items on receptacles
- **`doors.py`**: `DoorGenerator` - Door placement between rooms
- **`windows.py`**: `WindowGenerator` - Window placement on walls
- **`walls.py`**: `WallGenerator` - Wall structure generation

#### Asset Management
- **`objaverse_retriever.py`**: `ObjathorRetriever` - CLIP-based asset retrieval from Objaverse
- **`layers.py`**: Asset-to-rendering-layer mapping
- **`utils.py`**: Scene utilities, top-down image generation, video creation

#### Constraint Solving
- **`milp_utils.py`**: Mixed Integer Linear Programming solver for object placement
- **DFS Solver**: Default depth-first search solver (more reliable than MILP)

## Key Dependencies & Setup

### Environment Setup
```bash
# Python 3.10 required
conda create --name holodeck python=3.10
conda activate holodeck
pip install -r requirements.txt

# Specific AI2-THOR version required
pip install --extra-index-url https://ai2thor-pypi.allenai.org ai2thor==0+8524eadda94df0ab2dbb2ef5a577e4d37c712897
```

### Required Data Downloads
```bash
python -m objathor.dataset.download_holodeck_base_data --version 2023_09_23
python -m objathor.dataset.download_assets --version 2023_09_23
python -m objathor.dataset.download_annotations --version 2023_09_23
python -m objathor.dataset.download_features --version 2023_09_23
```

### Critical Dependencies
- **langchain-litellm**: LLM interface (supports OpenRouter, OpenAI)
- **open-clip-torch**: CLIP model for visual-semantic matching
- **sentence-transformers**: Text embedding for asset retrieval
- **ai2thor**: 3D simulation environment
- **objathor**: Asset management system
- **cvxpy/gurobipy**: Optimization solvers for constraint satisfaction
- **shapely**: Geometric computations
- **compress-json**: Scene file storage

## Development Commands

### Basic Usage
```bash
# Generate single scene
python ai2holodeck/main.py --query "a living room" --openai_api_key <KEY>

# Use OpenRouter instead of OpenAI
python ai2holodeck/main.py --query "a living room" --llm_model openrouter/openai/gpt-4.1-mini

# Debug with debugpy
debugpy ai2holodeck/main.py --query "a living room" --llm_model openrouter/openai/gpt-4.1-mini

# Generate variants of existing scene  
python ai2holodeck/main.py --mode generate_variants --original_scene path/to/scene.json --query "variant description"

# Batch generation from file
python ai2holodeck/main.py --mode generate_multi_scenes --query_file ./data/queries.txt
```

### Important Arguments
- `--use_milp False`: Use DFS solver instead of MILP (recommended for better layouts)
- `--random_selection True/False`: Control object selection diversity vs precision
- `--use_constraint True/False`: Enable/disable placement constraints
- `--single_room True`: Generate single-room scenes only
- `--generate_image True/False`: Create top-down view images
- `--generate_video True/False`: Create walk-through videos
- `--add_ceiling True/False`: Include ceiling objects
- `--used_assets path/to/exclusions.txt`: Exclude specific assets

### Development Tools
```bash
# Code formatting
make black

# Install for development
make install-dev

# Profile performance
make profile
```

## Configuration & Environment Variables

### Key Constants (`/home/ycho358/GitHub/Holodeck-Jae/ai2holodeck/constants.py`)
- **`OBJATHOR_ASSETS_BASE_DIR`**: Asset storage location (default: `~/.objathor-assets`)
- **`LLM_MODEL_NAME`**: Default LLM model (`gpt-4o-2024-05-13`)
- **`THOR_COMMIT_ID`**: Specific AI2-THOR commit for compatibility
- **`DEBUGGING`**: Enable debug mode via environment variable

### Environment Variables
- **`OPENAI_API_KEY`**: OpenAI API access
- **`OPENAI_ORG`**: OpenAI organization (optional)
- **`OBJAVERSE_ASSETS_DIR`**: Override asset directory location
- **`ASSETS_VERSION`**: Asset version (default: `2023_09_23`)
- **`DEBUGGING`**: Enable debug output

## Scene Structure & Data Flow

### Scene JSON Schema
Generated scenes follow a hierarchical structure:
- **`rooms`**: Rectangular room definitions with materials
- **`walls`**: Wall segments with materials and dimensions  
- **`doors`**: Door connections between rooms
- **`windows`**: Window placements on walls
- **`objects`**: All placed objects (floor + wall + small objects)
- **`proceduralParameters`**: Lighting, ceiling materials, skybox
- **`metadata`**: Agent positions and scene configuration

### Data Persistence
- **Output Location**: `./data/scenes/{query_name}-{timestamp}/`
- **Scene File**: `{query_name}.json` (compressed JSON format)
- **Visualization**: `{query_name}.png` (top-down view)
- **Video**: `{query_name}.mp4` (optional walk-through)

## Unity Integration

### Loading Scenes in Unity
1. **Unity Version**: `2020.3.25f1` (critical requirement)
2. **AI2-THOR Setup**: 
   ```bash
   git clone https://github.com/allenai/ai2thor.git
   git checkout 07445be8e91ddeb5de2915c90935c4aef27a241d
   ```
3. **Flask/Werkzeug Compatibility**:
   ```bash
   pip install Werkzeug==2.0.1 Flask==2.0.1
   ```
4. **Load Scene**:
   ```bash
   python connect_to_unity.py --scene path/to/scene.json
   ```

## Key Design Patterns & Workflows

### LLM Prompt Engineering
The system uses sophisticated prompt templates in `/home/ycho358/GitHub/Holodeck-Jae/ai2holodeck/generation/prompts.py`:
- **Structured output**: Prompts enforce specific formats for parsing
- **Context injection**: Room sizes, existing objects, and constraints
- **Multi-step reasoning**: Breaking complex placement into constraint chains

### Asset Retrieval Strategy
1. **CLIP-based matching**: Visual-semantic similarity between text and 3D assets
2. **Embedding caching**: Pre-computed features for fast retrieval  
3. **Exclusion lists**: Avoid duplicate assets across generations
4. **Size-aware selection**: Consider room dimensions for appropriate scaling

### Constraint Satisfaction
- **DFS Solver** (default): Iterative placement with collision avoidance
- **MILP Solver** (experimental): Mathematical optimization for complex constraints  
- **Layered constraints**: Global position → distance → alignment → rotation
- **Receptacle-aware**: Small objects placed on appropriate surfaces

### Multi-processing Architecture
- **Asset retrieval**: Parallel CLIP inference for object matching
- **Scene generation**: Memory-managed controllers to prevent leaks
- **Batch processing**: Queue-based multi-scene generation

## Common Issues & Solutions

### Performance Optimization
- Use DFS solver (`--use_milp False`) for reliable layouts
- Enable GPU for CLIP model processing (`device="cuda"`)  
- Limit concurrent processes for memory management
- Cache embeddings to avoid recomputation

### Asset Management
- Verify data download completion before generation
- Check `OBJATHOR_ASSETS_DIR` environment variable
- Ensure sufficient disk space for asset storage
- Monitor asset exclusion lists to maintain diversity

### LLM Integration
- GPT-4o-2024-05-13 access required for best results
- OpenRouter provides cost-effective alternative models
- Implement retry logic for API failures
- Monitor token usage for cost optimization

### Scene Quality
- Balance `random_selection` vs precision based on use case
- Use `single_room` flag for focused generation
- Adjust `retrieval_threshold` for asset matching sensitivity
- Enable constraints for realistic object placement

## Development Best Practices

### Code Organization
- **Modular generators**: Each component (rooms, objects, etc.) is self-contained
- **Prompt separation**: All LLM prompts centralized in `prompts.py`
- **Configuration isolation**: Constants and paths in dedicated file
- **Error handling**: Graceful degradation when components fail

### Testing & Debugging
- **Scene validation**: Check generated JSON structure before rendering
- **Visual debugging**: Use top-down images to verify layouts
- **Memory monitoring**: Watch for controller leaks in batch processing
- **Asset verification**: Confirm asset retrieval before placement

### Extension Points
- **Custom generators**: Follow existing generator patterns for new object types  
- **LLM backends**: Swap models via `llm_model_name` parameter
- **Constraint types**: Extend constraint vocabulary in prompts
- **Asset databases**: Integrate additional 3D asset collections

## Performance Characteristics

### Generation Time
- **Single room**: 2-5 minutes depending on complexity
- **Multi-room scenes**: 5-15 minutes with full object placement
- **Batch processing**: Scales linearly with scene count
- **Asset retrieval**: Dominated by CLIP inference time

### Resource Requirements  
- **Memory**: 8-16GB RAM for CLIP models and scene generation
- **GPU**: CUDA-capable GPU strongly recommended for CLIP
- **Storage**: 10-50GB for full asset database
- **Network**: Stable connection for LLM API calls

This architecture enables flexible, language-driven 3D scene generation with sophisticated constraint handling and asset management. The modular design supports easy extension and customization for specific use cases.