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


I'll create a comprehensive automated test case generator for Python files. This solution will analyze your Python files and generate test cases automatically.I've created a comprehensive automated test case generator for Python files. Here's also a simple shell script to make it even easier to use:Let me also create a simple example to demonstrate how the test generator works:And here's what the generated test file would look like:## How to Use the Test Generator

### Setup:

1. **Save the main script** (`test_generator.py`) to your system
2. **Make it executable** (optional):
   ```bash
   chmod +x test_generator.py
   ```

### Usage Examples:

**1. Generate tests for a single file:**
```bash
python test_generator.py /path/to/your/module.py
```

**2. Generate tests for multiple files:**
```bash
python test_generator.py /home/user/project/module1.py /home/user/project/module2.py
```

**3. Generate tests in a specific output directory:**
```bash
python test_generator.py -o tests/ /path/to/your/module.py
```

**4. Recursively generate tests for all Python files in a directory:**
```bash
python test_generator.py -r /path/to/your/project/
```

**5. Use the shell wrapper for convenience:**
```bash
# Save the shell script as 'generate-tests'
chmod +x generate-tests
./generate-tests /path/to/module.py
```

### Features:

1. **Automatic Analysis**: Parses Python files using AST to understand structure
2. **Smart Test Generation**: Creates tests for all public functions and class methods
3. **Intelligent Test Data**: Generates appropriate test values based on parameter names
4. **Edge Case Testing**: Includes basic edge case tests (like None values)
5. **Flexible Output**: Can save tests in the same directory or a custom location
6. **Batch Processing**: Process multiple files or entire directories at once
7. **Clean Structure**: Follows unittest framework conventions

### What the Generator Does:

1. **Analyzes** your Python files to find all functions and classes
2. **Generates** test methods with proper setup (Arrange-Act-Assert pattern)
3. **Creates** intelligent test data based on parameter names
4. **Saves** test files with proper naming convention (`test_<original_name>.py`)
5. **Handles** imports and path setup automatically

### Customization:

After generation, you should:
1. Review the generated tests
2. Add specific assertions based on your expected behavior
3. Customize test data for your specific use cases
4. Add additional edge cases or specific scenarios

The generator provides a solid foundation that you can build upon, saving you the time of writing boilerplate test code while allowing you to focus on the specific test logic for your application.

### test_generator.py:

#!/usr/bin/env python3
"""
Automated Test Case Generator for Python Files
Usage: python test_generator.py <file1.py> <file2.py> ...
"""

import ast
import os
import sys
import argparse
import inspect
import importlib.util
from pathlib import Path
from typing import List, Dict, Any, Tuple
import textwrap

class TestCaseGenerator:
    def __init__(self):
        self.test_template = """import unittest
import sys
from pathlib import Path
{imports}

# Add the directory containing the module to the path
sys.path.insert(0, str(Path('{module_path}').parent))

from {module_name} import *

class Test{class_name}(unittest.TestCase):
{test_methods}

if __name__ == '__main__':
    unittest.main()
"""

    def parse_python_file(self, file_path: str) -> Tuple[ast.Module, str]:
        """Parse a Python file and return its AST and content."""
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        tree = ast.parse(content)
        return tree, content

    def extract_functions_and_classes(self, tree: ast.Module) -> Dict[str, Any]:
        """Extract all functions and classes from the AST."""
        items = {
            'functions': [],
            'classes': []
        }
        
        for node in ast.walk(tree):
            if isinstance(node, ast.FunctionDef):
                # Skip private and magic methods for top-level functions
                if not node.name.startswith('_'):
                    func_info = {
                        'name': node.name,
                        'args': [arg.arg for arg in node.args.args],
                        'defaults': [ast.unparse(d) for d in node.args.defaults],
                        'return_annotation': ast.unparse(node.returns) if node.returns else None,
                        'docstring': ast.get_docstring(node),
                        'decorators': [ast.unparse(d) for d in node.decorator_list]
                    }
                    items['functions'].append(func_info)
            
            elif isinstance(node, ast.ClassDef):
                class_info = {
                    'name': node.name,
                    'methods': [],
                    'docstring': ast.get_docstring(node)
                }
                
                for item in node.body:
                    if isinstance(item, ast.FunctionDef) and not item.name.startswith('_'):
                        method_info = {
                            'name': item.name,
                            'args': [arg.arg for arg in item.args.args],
                            'docstring': ast.get_docstring(item)
                        }
                        class_info['methods'].append(method_info)
                
                items['classes'].append(class_info)
        
        return items

    def generate_test_value(self, arg_name: str, type_hint: str = None) -> str:
        """Generate appropriate test values based on argument name or type hint."""
        # Common patterns for argument names
        if any(x in arg_name.lower() for x in ['name', 'title', 'label', 'text', 'message']):
            return '"test_string"'
        elif any(x in arg_name.lower() for x in ['count', 'number', 'num', 'size', 'length']):
            return '10'
        elif any(x in arg_name.lower() for x in ['flag', 'is_', 'has_', 'enabled']):
            return 'True'
        elif any(x in arg_name.lower() for x in ['list', 'array', 'items']):
            return '[1, 2, 3]'
        elif any(x in arg_name.lower() for x in ['dict', 'map', 'config']):
            return '{"key": "value"}'
        elif any(x in arg_name.lower() for x in ['path', 'file', 'dir']):
            return '"/tmp/test_file.txt"'
        elif arg_name.lower() in ['x', 'y', 'z', 'a', 'b', 'c']:
            return '5'
        else:
            return '"test_value"'

    def generate_function_test(self, func_info: Dict[str, Any]) -> str:
        """Generate test method for a function."""
        func_name = func_info['name']
        args = func_info['args']
        
        # Skip 'self' if present
        args = [arg for arg in args if arg != 'self']
        
        # Generate test values for arguments
        test_args = []
        for arg in args:
            test_args.append(self.generate_test_value(arg))
        
        test_method = f"""
    def test_{func_name}_basic(self):
        \"\"\"Test {func_name} with basic inputs.\"\"\"
        # Arrange
        {', '.join(args) if args else 'pass'} = {', '.join(test_args) if test_args else 'None'}
        
        # Act
        result = {func_name}({', '.join(args)})
        
        # Assert
        self.assertIsNotNone(result)  # Basic assertion - customize based on expected behavior
        
    def test_{func_name}_edge_cases(self):
        \"\"\"Test {func_name} with edge cases.\"\"\"
        # Test with None values (if applicable)
        try:
            result = {func_name}({', '.join(['None' for _ in args])})
        except Exception as e:
            # Expected behavior for None inputs
            pass
"""
        return test_method

    def generate_class_test(self, class_info: Dict[str, Any]) -> str:
        """Generate test methods for a class."""
        class_name = class_info['name']
        test_methods = [f"""
    def test_{class_name.lower()}_instantiation(self):
        \"\"\"Test {class_name} instantiation.\"\"\"
        instance = {class_name}()
        self.assertIsInstance(instance, {class_name})
"""]
        
        for method in class_info['methods']:
            if method['name'] != '__init__':
                method_args = [arg for arg in method['args'] if arg != 'self']
                test_args = [self.generate_test_value(arg) for arg in method_args]
                
                test_methods.append(f"""
    def test_{class_name.lower()}_{method['name']}(self):
        \"\"\"Test {class_name}.{method['name']} method.\"\"\"
        # Arrange
        instance = {class_name}()
        {', '.join(method_args) if method_args else 'pass'} = {', '.join(test_args) if test_args else 'None'}
        
        # Act
        result = instance.{method['name']}({', '.join(method_args)})
        
        # Assert
        # Add specific assertions based on expected behavior
        pass
""")
        
        return '\n'.join(test_methods)

    def generate_test_file(self, file_path: str, items: Dict[str, Any]) -> str:
        """Generate complete test file content."""
        module_path = os.path.abspath(file_path)
        module_name = Path(file_path).stem
        
        # Generate imports
        imports = []
        
        # Generate test methods
        test_methods = []
        
        # Generate tests for functions
        for func in items['functions']:
            test_methods.append(self.generate_function_test(func))
        
        # Generate tests for classes
        for cls in items['classes']:
            test_methods.append(self.generate_class_test(cls))
        
        if not test_methods:
            test_methods = ["""
    def test_placeholder(self):
        \"\"\"Placeholder test - no testable items found.\"\"\"
        self.assertTrue(True)
"""]
        
        # Create class name from module name
        class_name = ''.join(word.capitalize() for word in module_name.split('_'))
        
        return self.test_template.format(
            imports='\n'.join(imports),
            module_path=module_path,
            module_name=module_name,
            class_name=class_name,
            test_methods='\n'.join(test_methods)
        )

    def save_test_file(self, test_content: str, original_path: str, output_dir: str = None) -> str:
        """Save the generated test file."""
        original_path = Path(original_path)
        
        if output_dir:
            # Save to specified output directory
            output_path = Path(output_dir)
            output_path.mkdir(parents=True, exist_ok=True)
            test_file_path = output_path / f"test_{original_path.name}"
        else:
            # Save in the same directory as the original file
            test_file_path = original_path.parent / f"test_{original_path.name}"
        
        with open(test_file_path, 'w', encoding='utf-8') as f:
            f.write(test_content)
        
        return str(test_file_path)

    def generate_tests_for_file(self, file_path: str, output_dir: str = None) -> str:
        """Generate tests for a single Python file."""
        try:
            print(f"Analyzing {file_path}...")
            tree, content = self.parse_python_file(file_path)
            items = self.extract_functions_and_classes(tree)
            
            print(f"Found {len(items['functions'])} functions and {len(items['classes'])} classes")
            
            test_content = self.generate_test_file(file_path, items)
            test_file_path = self.save_test_file(test_content, file_path, output_dir)
            
            print(f"Generated test file: {test_file_path}")
            return test_file_path
            
        except Exception as e:
            print(f"Error processing {file_path}: {str(e)}")
            return None

def main():
    parser = argparse.ArgumentParser(
        description='Generate test cases for Python files',
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=textwrap.dedent('''
        Examples:
            # Generate tests for a single file
            python test_generator.py /path/to/module.py
            
            # Generate tests for multiple files
            python test_generator.py /path/to/file1.py /path/to/file2.py
            
            # Generate tests in a specific output directory
            python test_generator.py -o tests/ /path/to/module.py
            
            # Generate tests for all Python files in a directory
            python test_generator.py -r /path/to/project/
        ''')
    )
    
    parser.add_argument('files', nargs='*', help='Python files to generate tests for')
    parser.add_argument('-o', '--output', help='Output directory for test files')
    parser.add_argument('-r', '--recursive', help='Recursively find Python files in directory')
    parser.add_argument('-v', '--verbose', action='store_true', help='Verbose output')
    
    args = parser.parse_args()
    
    generator = TestCaseGenerator()
    files_to_process = []
    
    # Handle recursive directory processing
    if args.recursive:
        root_path = Path(args.recursive)
        if root_path.is_dir():
            files_to_process.extend([str(f) for f in root_path.rglob('*.py') 
                                   if not f.name.startswith('test_')])
        else:
            print(f"Error: {args.recursive} is not a directory")
            sys.exit(1)
    
    # Add individual files
    files_to_process.extend(args.files)
    
    if not files_to_process:
        parser.print_help()
        sys.exit(1)
    
    # Generate tests for each file
    generated_files = []
    for file_path in files_to_process:
        if os.path.exists(file_path):
            result = generator.generate_tests_for_file(file_path, args.output)
            if result:
                generated_files.append(result)
        else:
            print(f"Warning: File not found - {file_path}")
    
    print(f"\nTest generation complete! Generated {len(generated_files)} test files.")
    
    if generated_files and args.verbose:
        print("\nGenerated test files:")
        for f in generated_files:
            print(f"  - {f}")

if __name__ == "__main__":
    main()

### test_generator_shell_script.py:

#!/bin/bash

# Test Generator Wrapper Script
# Save this as 'generate-tests' and make it executable with: chmod +x generate-tests

# Color codes for output
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

# Function to display usage
usage() {
    echo "Usage: $0 [OPTIONS] <python_files...>"
    echo ""
    echo "Options:"
    echo "  -o, --output DIR     Output directory for test files"
    echo "  -r, --recursive DIR  Recursively find Python files in directory"
    echo "  -h, --help          Show this help message"
    echo ""
    echo "Examples:"
    echo "  $0 /path/to/module.py"
    echo "  $0 -o tests/ /path/to/file1.py /path/to/file2.py"
    echo "  $0 -r /path/to/project/"
}

# Check if Python is installed
if ! command -v python3 &> /dev/null; then
    echo -e "${RED}Error: Python 3 is not installed${NC}"
    exit 1
fi

# Save the test generator script if it doesn't exist
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" &> /dev/null && pwd )"
TEST_GENERATOR_PATH="$SCRIPT_DIR/test_generator.py"

if [ ! -f "$TEST_GENERATOR_PATH" ]; then
    echo -e "${YELLOW}Test generator script not found. Creating it...${NC}"
    # You would need to create the test_generator.py file here
    # For now, we'll assume it exists in the same directory
fi

# Parse command line arguments
if [ $# -eq 0 ]; then
    usage
    exit 1
fi

# Pass all arguments to the Python script
echo -e "${GREEN}Starting test generation...${NC}"
python3 "$TEST_GENERATOR_PATH" "$@"

# Check exit status
if [ $? -eq 0 ]; then
    echo -e "${GREEN}Test generation completed successfully!${NC}"
else
    echo -e "${RED}Test generation failed!${NC}"
    exit 1
fi
