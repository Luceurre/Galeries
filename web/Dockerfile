FROM python:3-jessie
MAINTAINER Pierre Glandon

ENV FLASK_APP=ponthe
ENV PROD_MODE=true

RUN apt-get update && apt-get install -y redis-server mysql-client
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        ssh \
        libssl-dev \
        pkg-config && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV RUSTUP_HOME=/rust
ENV CARGO_HOME=/cargo
ENV PATH=/cargo/bin:/rust/bin:$PATH

RUN echo "(curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain nightly --no-modify-path) && rustup default nightly" > /install-rust.sh && chmod 755 /install-rust.sh

WORKDIR /app

COPY ./app/requirements.txt .
RUN pip install --upgrade pip
RUN pip install -r requirements.txt

COPY ./app /app

RUN mkdir -p /app/instance/static/uploads
RUN mkdir -p /app/instance/tmp/uploads
RUN mkdir -p /app/instance/static/thumbs

RUN cd /app/instance/keys && openssl genrsa -out jwtRS256-private.pem 2048 && openssl rsa -in jwtRS256-private.pem -pubout -out jwtRS256-public.pem

ENV PYTHONPATH /app
#CMD flask run --host=0.0.0.0:7000
CMD gunicorn --bind 0.0.0.0:7000 wsgi:app