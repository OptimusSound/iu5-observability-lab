FROM python:3.12-slim-bullseye
COPY ./requirements.txt /app/requirements.txt
WORKDIR /app
RUN pip install --no-cache-dir --progress-bar off -r requirements.txt
RUN mkdir -p /app/log
COPY app.py /app
EXPOSE 5000
CMD ["python", "app.py" ]
