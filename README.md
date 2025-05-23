## Pynguin

# Create a virtual environment (recommended)
python -m venv pynguin_env

# Activate the virtual environment
# On macOS/Linux:
source pynguin_env/bin/activate
# On Windows:
# pynguin_env\Scripts\activate

# Install Pynguin
pip install pynguin


PYNGUIN_DANGER_AWARE=true python -m pynguin --project_path path/to/your/project --module-name calculator --output-path tests/
pytest tests/ -v
