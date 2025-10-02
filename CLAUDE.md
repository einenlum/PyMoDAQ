# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

PyMoDAQ (Modular Data Acquisition with Python) is a framework for interfacing scientific experiments with detectors and actuators. It provides both a complete dashboard interface for automated measurements and modular tools for building custom applications.

## Development Setup

Install in editable mode with development dependencies:
```bash
pip install -e ".[dev]"
```

On Linux, Qt backends require additional system packages:
```bash
sudo apt install -y libxkbcommon-x11-0 libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-cursor0 libxcb-randr0 libxcb-render-util0 libxcb-xinerama0 libxcb-xfixes0 x11-utils libgl1 libegl1
```

Install one of the Qt backends:
```bash
pip install pyqt5  # or pyqt6 or pyside6
```

## Testing

Run all tests:
```bash
pytest -vv --cov=pymodaq -n 1
```

Run a single test file:
```bash
pytest -vv tests/path/to/test_file.py
```

Run a specific test:
```bash
pytest -vv tests/path/to/test_file.py::test_name
```

The test suite runs against all Python versions (3.9-3.12) and all Qt backends (PyQt5, PyQt6, PySide6) in CI.

## Linting

Check for critical errors:
```bash
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics --exclude=docs
```

## Architecture

### Core Structure

**src/pymodaq/** - Main package organized by:
- **control_modules/** - Core instrument control modules:
  - `daq_move.py` - Actuator control module (DAQ_Move)
  - `daq_viewer.py` - Detector control module (DAQ_Viewer)
  - `*_utility_classes.py` - Base classes for instrument plugins

- **extensions/** - Dashboard extensions:
  - `daq_scan.py` - Automated acquisition scanning
  - `daq_logger/` - Data logging to HDF5/SQL
  - `h5browser.py` - HDF5 file browser
  - `pid/` - PID control
  - `bayesian/` - Bayesian optimization

- **utils/** - Supporting utilities:
  - `managers/` - Manager classes (ModulesManager, PresetManager, RemoteManager, etc.)
  - `scanner/` - Scanning modes and utilities
  - `h5modules/` - HDF5 file handling
  - `leco/` - LECO protocol support for remote control
  - `parameter/` - Parameter tree utilities
  - `gui_utils/` - GUI helper functions

- **dashboard.py** - Main Dashboard application that coordinates control modules and extensions

### Plugin System

PyMoDAQ uses a plugin architecture for hardware instruments. Plugins are separate packages that follow the naming convention `pymodaq_plugins_*`.

- Instrument plugins inherit from `DAQ_Move_base` (actuators) or `DAQ_Viewer_base` (detectors)
- Viewers are categorized by dimensionality: 0D, 1D, or 2D
- Use the [template repository](https://github.com/PyMoDAQ/pymodaq_plugins_template) as a starting point
- The `pymodaq_plugins_mock` package provides mock instruments for testing
- Plugins are discovered automatically at startup

### Key Design Patterns

- **Factory Pattern**: Used extensively for extensibility (data exporters, ROI math functions, scanning modes, plotters)
- **Manager Pattern**: Centralized management classes for modules, presets, overshoots, and remote connections
- **Signal/Slot**: Qt-based event system for UI and module communication
- **Plugin Discovery**: Dynamic loading of instrument plugins from installed packages

### Data Flow

1. **Control Modules** (DAQ_Move/DAQ_Viewer) interface with hardware via plugins
2. **Dashboard** orchestrates multiple control modules
3. **Extensions** use control modules for advanced functionality (scanning, logging, PID)
4. Data is handled through `DataToExport` and `DataActuator` classes
5. HDF5 is the primary storage format via `h5modules`

## Branch Structure

- **5.0.x** - Current stable branch (production releases like 5.0.0, 5.0.1)
- **5.1.x_dev** - Development branch for next minor version
- **feature/*** - New features (branch from development)
- **bugfix/*** - Bug fixes (can branch from stable or development)
- **hotfix/*** - Critical bugs requiring immediate release

Pull requests for bug fixes must be applied to all affected branches. Hotfixes trigger immediate patch releases.

## Configuration and Initialization

- PyMoDAQ creates a local configuration directory at `/etc/.pymodaq` on Linux
- Config handling via `pymodaq_utils.config.Config`
- Preset configurations stored and managed by `PresetManager`
- Initialization in `__init__.py` registers scanners and discovers plugins
- Uses `hatch` for build system with version from VCS tags

## Entry Points

Console scripts defined in `pyproject.toml`:
- `dashboard` - Main Dashboard application
- `daq_move` - Standalone actuator control
- `daq_viewer` - Standalone detector control
- `daq_scan` - Scanning extension
- `daq_logger` - Data logging extension
- `h5browser` - HDF5 browser
- `pymodaq_updater` - Plugin manager/updater

## Repository Structure

PyMoDAQ is split into four separate repositories, each serving a distinct purpose:

### 1. **pymodaq_utils** (Foundation Layer)
**Repository:** https://github.com/PyMoDAQ/pymodaq_utils
**Version:** 0.0.x (stable branch: 0.0.x_dev)
**Purpose:** Core utilities and base classes shared across all PyMoDAQ packages

**Key Components:**
- `config.py` - Configuration management (Config class)
- `logger.py` - Logging utilities
- `enums.py` - Common enumerations
- `factory.py` - Factory pattern implementation
- `math_utils.py` - Mathematical utilities
- `units.py` - Physical units handling with pint
- `serialize/` - Data serialization for network communication
- `array_manipulation.py` - Array and data manipulation utilities
- `environment.py` - Environment detection and setup

**Dependencies:** Only fundamental packages (numpy, scipy, pint, multipledispatch, toml)
**Role:** Provides the foundation layer with no Qt dependencies; used by all other packages

### 2. **pymodaq_data** (Data Management Layer)
**Repository:** https://github.com/PyMoDAQ/pymodaq_data
**Version:** 5.0.x (stable branch: 5.0.x_dev)
**Purpose:** Data structures and HDF5 file management

**Key Components:**
- `data.py` - Core data classes (Data0D, Data1D, Data2D, DataND, DataToExport, DataActuator)
- `h5modules/` - HDF5 file handling:
  - `data_saving.py` - Save/load data to/from HDF5
  - `backends.py` - HDF5 backend abstraction (h5py, tables)
  - `browsing.py` - Browse HDF5 file structure
  - `exporters/` - Export to other formats (HyperSpy, FLIMJ)
- `plotting/plotter/` - Plotting abstraction layer (matplotlib backend)
- `post_treatment/` - Data post-processing utilities
- `numpy_func.py` - NumPy helper functions

**Dependencies:** `pymodaq_utils`, numpy, scipy, pint, tables/h5py
**Role:** Handles all data objects with metadata, units, axes, and uncertainty; manages HDF5 persistence

### 3. **pymodaq_gui** (GUI Layer)
**Repository:** https://github.com/PyMoDAQ/pymodaq_gui
**Version:** 5.0.x (stable branch: 5.0.x_dev)
**Purpose:** Qt-based graphical user interface components

**Key Components:**
- `parameter/` - Parameter tree system (based on pyqtgraph's ParameterTree)
- `plotting/data_viewers/` - Data visualization widgets (Viewer0D, Viewer1D, Viewer2D, ViewerND)
- `utils/` - GUI utilities:
  - `dock.py` - Dockable windows (Dock, DockArea)
  - `widgets/` - Custom Qt widgets (LCD, tables, etc.)
  - `splash.py` - Splash screen
  - `custom_app.py` - Custom Qt application
  - `file_io.py` - File selection dialogs
- `h5modules/` - GUI components for HDF5 browsing
- `managers/` - GUI-related managers:
  - `parameter_manager.py` - Parameter tree management
  - `action_manager.py` - Action/menu management
  - `roi_manager.py` - Region of interest management
- `messenger.py` - Message boxes and user notifications

**Dependencies:** `pymodaq_utils`, `pymodaq_data`, qtpy, pyqtgraph
**Entry Points:** `h5browser`, `parameter_example`
**Role:** Provides all Qt-based UI components; visualizes data objects from pymodaq_data

### 4. **PyMoDAQ** (Main Application Layer)
**Repository:** https://github.com/PyMoDAQ/PyMoDAQ (this repository)
**Version:** 5.0.x (stable branch: 5.0.x)
**Purpose:** Main application framework, control modules, and extensions

**Key Components:** See "Core Structure" section below
**Dependencies:** All three packages above plus application-specific libraries
**Entry Points:** `dashboard`, `daq_move`, `daq_viewer`, `daq_scan`, `daq_logger`, `h5browser`, `pymodaq_updater`
**Role:** Orchestrates everything; provides the main Dashboard, control modules, and extensions

## Dependency Chain

```
pymodaq_utils (foundation)
    ↓
pymodaq_data (data structures)
    ↓
pymodaq_gui (Qt UI components)
    ↓
PyMoDAQ (main application)
```

**Key Interactions:**
- **PyMoDAQ → pymodaq_gui**: Uses Parameter trees, Docks, data viewers, widgets
- **PyMoDAQ → pymodaq_data**: Uses DataToExport, DataActuator, HDF5 saving/loading
- **PyMoDAQ → pymodaq_utils**: Uses Config, logger, enums, factory, serialization
- **pymodaq_gui → pymodaq_data**: Visualizes data objects
- **pymodaq_gui → pymodaq_utils**: Uses config, logger, utilities
- **pymodaq_data → pymodaq_utils**: Uses utilities, units, config

## Local Development

The three dependency packages are cloned in `local_deps/`:
- `local_deps/pymodaq_utils/`
- `local_deps/pymodaq_data/`
- `local_deps/pymodaq_gui/`

To work with local development versions:
```bash
pip install -e local_deps/pymodaq_utils
pip install -e local_deps/pymodaq_data
pip install -e local_deps/pymodaq_gui
pip install -e ".[dev]"
```

## Additional Dependencies

**Application-level dependencies:**
- Qt abstraction via `qtpy` (supports PyQt5/6, PySide6)
- `pyqtgraph` - Plotting and visualization
- `h5py` - HDF5 file operations (optional dev dependency)
- `pint` - Physical units handling
- `scipy`, `numpy <2.0.0` - Scientific computing
- `pyleco` - Remote control protocol (LECO)
- `bayesian-optimization` - Optimization algorithms
- `pymodaq_plugin_manager` - Plugin discovery and management
- `pymodaq_plugins_mock` - Mock instruments for testing
