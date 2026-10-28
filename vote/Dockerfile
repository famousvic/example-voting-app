# base defines a base stage that uses the official python runtime base image
FROM python:3.11-slim@sha256:e9e0bd902ae57c30d3ffdb7019b33785cee3c7860dc1eda2dfd740354911a82c AS base

# Add curl for healthcheck
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Run as a non-root user
RUN useradd --system --uid 10001 --no-create-home app

# Set the application directory
WORKDIR /usr/local/app

# Install our requirements.txt
COPY requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# dev defines a stage for development, where it'll watch for filesystem changes
FROM base AS dev
RUN pip install watchdog
ENV FLASK_ENV=development
USER app
CMD ["python", "app.py"]

# final defines the stage that will bundle the application for production
FROM base AS final

# Copy our code from the current folder to the working directory inside the container
COPY --chown=app:app . .

# Make port 80 available for links and/or publish
EXPOSE 8000

USER app

# Define our command to be run when launching the container
CMD ["gunicorn", "app:app", "-b", "0.0.0.0:8000", "--log-file", "-", "--access-logfile", "-", "--workers", "4", "--keep-alive", "0"]
