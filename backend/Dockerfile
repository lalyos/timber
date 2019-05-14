FROM python:3.7-alpine3.8

WORKDIR /app

RUN apk --no-cache add curl git bash

COPY requirements.txt /app
RUN pip install --no-cache-dir -r requirements.txt
COPY . /app

CMD /app/start

# prod -> gunicorn
ENV PRODUCTION=true
ENV TIMBER_LISTEN_PORT=8080

#ENV TIMBER_ACCESS_LOG=access.log # default: stdout
#ENV TIMBER_LOG_FORMAT=json # default: line based text
#ENV GUNICORN_ARGS=--statsd-host=graphite:8125 --statsd-prefix=$HOSTNAME

EXPOSE 8080
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1
