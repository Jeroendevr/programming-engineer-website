# Python Project Setup and Layout


```{note}
Author: Jeroen de Vries  
Date Posted: 23-may-2023
```

When working with Python project I sometimes stuble across some issues with my specific setup. So hereby a guide how I set-up my python projects

# Prerequisite
The following packages and tools are default for me so it should work with the following:
- Test framework - Pytest
  - I configure pytest using the pyproject.toml files, therefore it is included in the layout
- Editor/IDE - Visual Studio Code
- Flask
- Shell

# Goals
Providing a setup that can be used for development as well deploying on (serverless) platforms. 

# Issues to adress
- Importing the project modules in differtent parts of your code.
- Invoking tests from within the IDE as using the Shell

# Layout
Both Flask and Pytest have some suggestion how a project layout should look like. In order to make it easier to compare the layouts. Every project has the same folders and files. 
- `project_folder`
- `tests`

## Flask layout
```
project_root/
├── project_folder
├── tests
└── pyproject.toml
```

## Pytest layout
```
project_root/
├── project_folder
│   └── src
├── tests
└── pyproject.toml
```

# Testing
In order to test the package install it first as an editable install.
`pip install --editable .`
Can check with
`pytest --collect-only` if the test can be collected correctly.  

After installation the package itself can be reused for imports. 