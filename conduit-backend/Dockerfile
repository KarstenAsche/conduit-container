# Stage 1: Install Python dependencies.
FROM python:3.5.2 AS builder

WORKDIR /app

# Install dependencies into a separate directory for the runtime stage.
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# Stage 2: Run the Django application.
FROM python:3.5.2 AS runtime

WORKDIR /app

# Copy the installed dependencies from the builder stage.
COPY --from=builder /install/ /usr/local/

# Copy the application source code.
COPY . .

# Configure Python for container execution.
ENV PYTHONUNBUFFERED=1

# Expose the internal application port.
EXPOSE 8000

# Start Django through Gunicorn instead of the development server.
ENTRYPOINT ["gunicorn", "conduit.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "2", "--access-logfile", "-", "--error-logfile", "-"]
