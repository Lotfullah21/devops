## DJANGO_SETTINGS_MODULE=your_project.settings

Tells Django which settings file to use when the application starts
Critical for configuration management
Typically points to the project's main settings file
Allows switching between different settings (e.g., development, production, testing) easily

## PYTHONUNBUFFERED=1

Disables Python's output buffering
Ensures that log messages and print statements are immediately displayed
Crucial for container logging and debugging
Prevents output from being held in memory buffers
