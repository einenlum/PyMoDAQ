PyMoDAQ Monorepo
#################

.. image:: https://img.shields.io/pypi/v/pymodaq.svg
   :target: https://pypi.org/project/pymodaq/
   :alt: Latest Version

.. image:: https://readthedocs.org/projects/pymodaq/badge/?version=latest
   :target: https://pymodaq.readthedocs.io/en/stable/?badge=latest
   :alt: Documentation Status

.. image:: https://codecov.io/gh/PyMoDAQ/PyMoDAQ/graph/badge.svg?token=IQNJRCQDM2
   :target: https://codecov.io/gh/PyMoDAQ/PyMoDAQ


.. figure:: http://pymodaq.cnrs.fr/en/latest/_static/splash.png
   :alt: PyMoDAQ splash


PyMoDAQ, Modular Data Acquisition with Python, is a set of **python** modules used to interface any kind of experiments.
It simplifies the interaction with detector and actuator hardware to go straight to the data acquisition of interest.

Overview
========

This is a **monorepo** containing all PyMoDAQ packages. Starting from version 5.1.0, all packages share a unified version number
and are developed together in this repository, while still being published as separate PyPI packages.

Packages
========

This repository contains four packages:

* **pymodaq_utils** - Core utilities and base classes
* **pymodaq_data** - Data structures and HDF5 file management
* **pymodaq_gui** - Qt-based graphical user interface components
* **pymodaq** - Main application framework with Dashboard, control modules, and extensions

All packages are versioned together (currently 5.1.0) but published separately to PyPI.

Installation
============

For end users, install the main package:

.. code-block:: bash

   pip install pymodaq

This will automatically install all dependencies (pymodaq_utils, pymodaq_data, pymodaq_gui).

For developers:

.. code-block:: bash

   git clone https://github.com/PyMoDAQ/PyMoDAQ
   cd PyMoDAQ

   # Install all packages in editable mode
   pip install -e packages/pymodaq_utils
   pip install -e packages/pymodaq_data
   pip install -e packages/pymodaq_gui
   pip install -e "packages/pymodaq[dev]"

   # Install a Qt backend (choose one)
   pip install pyqt5  # or pyqt6 or pyside6

Key Features
============

* **Dashboard**: Complete interface for automated measurements without writing custom code
* **Control Modules**:

  - DAQ_Move: Control actuators/motors
  - DAQ_Viewer: Control detectors/cameras

* **Extensions**:

  - DAQ_Scan: Automated acquisition scanning
  - DAQ_Logger: Data logging to HDF5/SQL
  - H5Browser: Browse and analyze HDF5 files
  - PID Control: Closed-loop control
  - Bayesian Optimization: Intelligent parameter optimization

* **Plugin System**: Hardware drivers are separate packages (``pymodaq_plugins_*``)

Documentation
=============

Full documentation: http://pymodaq.cnrs.fr

Testing
=======

Run all tests across all packages:

.. code-block:: bash

   pytest -vv

Run tests for a specific package:

.. code-block:: bash

   cd packages/pymodaq_utils
   pytest -vv --cov=. -n 1

The test suite runs against Python 3.9-3.12 and all Qt backends (PyQt5, PyQt6, PySide6).

Contributing
============

Contributions are welcome! This monorepo structure allows you to make changes across multiple packages in a single PR.

1. Fork the repository
2. Create a feature branch
3. Make changes (can span multiple packages)
4. Run tests: ``pytest -vv``
5. Submit a pull request

License
=======

MIT License - See LICENSE file for details

Citation
========

If you use PyMoDAQ in your research, please cite:

.. code-block:: bibtex

   @article{pymodaq,
     title={PyMoDAQ: An open-source Python-based software for modular data acquisition},
     author={Weber, S{\'e}bastien J.},
     journal={Review of Scientific Instruments},
     volume={92},
     number={4},
     pages={045104},
     year={2021},
     publisher={AIP Publishing LLC}
   }
