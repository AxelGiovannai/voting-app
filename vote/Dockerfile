FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 80
# utilisateur non root
RUN useradd -m appuser
RUN chown -R appuser:appuser /app
USER appuser
CMD ["python", "app.py"]