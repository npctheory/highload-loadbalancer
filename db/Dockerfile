FROM postgres:latest
USER root

RUN apt-get update && \
    apt-get install -y python3 python3-pip python3-venv procps && \
    rm -rf /var/lib/apt/lists/*

RUN python3 -m venv /opt/venv && \
    /opt/venv/bin/pip install --upgrade pip && \
    /opt/venv/bin/pip install psycopg2-binary requests==2.31.0

ENV PATH="/opt/venv/bin:$PATH"

COPY pg_backup/pg_backup.tar.part* /tmp/
RUN cat /tmp/pg_backup.tar.part* > /pg_backup.tar && \
    tar -xf /pg_backup.tar -C /var/lib/postgresql/data && \
    chown -R 999:999 /var/lib/postgresql/data && \
    find /var/lib/postgresql/data -type d -exec chmod 755 {} \; && \
    find /var/lib/postgresql/data -type f -exec chmod 644 {} \;
	
COPY docker-healthcheck /docker-healthcheck/

USER postgres
ENTRYPOINT ["docker-entrypoint.sh"]
EXPOSE 5432
CMD ["postgres"]