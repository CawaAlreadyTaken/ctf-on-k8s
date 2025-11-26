# Base light image
FROM python:3.12-alpine

WORKDIR /app

# Non privileged user
RUN adduser -D nonroot

# Create /app/static and a test file
RUN mkdir -p /app/static \
    && echo "Hello from main-category-01-name" > /app/static/index.html

# Make /app non-writeable, but let /app/static writeable
RUN chown -R nonroot /app \
    && chmod a-w /app \
    && chmod u+w /app/static

# Go to non-root user
USER nonroot

EXPOSE 5000

# Start an HTTP server on /app/static
# python -m http.server serves from workdir
WORKDIR /app/static
CMD ["python", "-m", "http.server", "5000"]

