FROM python:3.8.0-alpine3.10

RUN mkdir -p /app
WORKDIR /app
COPY app ./
USER 1001:1001
EXPOSE 8000
ENTRYPOINT ["python3", "-m", "http.server"]
