FROM python:3.9-slim

WORKDIR /code

# Dépendances de build pour psycopg2/asyncpg 
RUN apt-get update \
  && apt-get install -y --no-install-recommends gcc libpq-dev python3-dev \
  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /code/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt

COPY ./app /code/app

# Uvicorn/Gunicorn exposent 80 dans les exemples du cours 
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
