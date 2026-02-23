FROM postgres:15

RUN apt-get update && apt-get install -y \
    python3 python3-pip python3-venv \
    libpq-dev gcc curl netcat-openbsd

RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

RUN pip install --upgrade pip
RUN pip install patroni[zookeeper] psycopg2-binary

COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml