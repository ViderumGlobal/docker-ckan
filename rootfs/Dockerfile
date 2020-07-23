FROM keitaro/base:0.5

MAINTAINER Keitaro Inc <info@keitaro.info>

ENV APP_DIR=/srv/app
ENV SRC_DIR=/srv/app/src
ENV PIP_SRC=${SRC_DIR}
ENV GIT_URL=https://github.com/ckan/ckan.git
ENV GIT_BRANCH=ckan-2.7.7
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS image_view text_view recline_view datastore datapusher archiver report envvars

WORKDIR ${APP_DIR}

#########################
### Base docker layer ###
#########################
# Install necessary packages to run CKAN

RUN apk add --no-cache git \
        gettext \
        postgresql-client \
        python \
        apache2-utils \
        libxml2 \
        libxslt \
        musl-dev \
        uwsgi \
        uwsgi-http \
        uwsgi-corerouter \
        uwsgi-python \
        py2-gevent \
        uwsgi-gevent \
        libmagic \
        varnish \
        sudo && \
    # Packages to build CKAN requirements and plugins
    apk add --no-cache --virtual .build-deps \
        postgresql-dev \
        gcc \
        make \
        g++ \
        autoconf \
        automake \
	libtool \
        python-dev \
        libxml2-dev \
        libxslt-dev \
        linux-headers && \
    # Create SRC_DIR
    mkdir -p ${SRC_DIR} && \
    # Create nginx run dir
    mkdir -p /run/nginx && \
    # Install pip, supervisord and uwsgi
    curl -o ${SRC_DIR}/get-pip.py https://bootstrap.pypa.io/get-pip.py && \
    python ${SRC_DIR}/get-pip.py && \
    pip install supervisor && \
    mkdir /etc/supervisord.d && \
    #pip wheel --wheel-dir=/wheels uwsgi gevent && \
    rm -rf ${SRC_DIR}/get-pip.py

COPY setup/supervisord.conf /etc

#RUN mkdir -p /var/log && \
#    mkdir -m 0644 -p /var/spool/cron/crontabs && \
#    touch /var/log/cron.log && \
#    mkdir -m 0644 -p /etc/cron.d

############################
### libgeos docker layer ###
############################
# Build and install libgeos to support geospatial
#ENV GEOS_VERSION=3.6.1
#RUN git clone -b ${GEOS_VERSION} --depth=1 --single-branch https://git.osgeo.org/gogs/geos/geos.git ${SRC_DIR}/geos && \
#    cd ${SRC_DIR}/geos && \
#    ./autogen.sh && \
#    ./configure --prefix /usr && \
#    make -j2 && \
#    make install-strip && \
#    rm -rf ${SRC_DIR}/geos

###########################
### redis docker layers ###
###########################
# Compile and install redis needed for harvest and CKAN 2.7
ENV REDIS_VERSION=3.2.9
ENV CKAN__HARVEST__MQ__TYPE=redis
ENV CKAN__HARVEST__MQ__HOSTNAME=localhost
ENV CKAN__HARVEST__MQ__PORT=6379
RUN apk add --no-cache linux-headers && \
    cd ${SRC_DIR} && \
    wget http://download.redis.io/releases/redis-${REDIS_VERSION}.tar.gz && \
    tar xzf redis-${REDIS_VERSION}.tar.gz && \
    rm -f redis-${REDIS_VERSION}.tar.gz && \
    cd redis-${REDIS_VERSION} && \
    make && \
    cp src/redis-server /bin/redis-server && \
    rm -rf ${SRC_DIR}/redis-${REDIS_VERSION}

#########################
### Celery docker layer ###
#########################
# Install Celery
RUN pip install celery==3.1.0 && \
    pip install kombu==3.0.30 && \
    pip install redis

#########################
### CKAN docker layer ###
#########################
# Install CKAN
RUN pip install -e git+${GIT_URL}@${GIT_BRANCH}#egg=ckan && \
    cd ${SRC_DIR}/ckan && \
    cp who.ini ${APP_DIR} && \
    sed -i 's/psycopg2==2.4.5/psycopg2==2.7.3.2/' requirements.txt && \
    pip install -r requirements.txt && \
    # Install CKAN envvars to support loading config from environment variables
    pip install -e git+https://github.com/okfn/ckanext-envvars.git#egg=ckanext-envvars && \
    pip install -e git+http://github.com/datagovuk/ckanext-report.git#egg=ckanext-report && \
    pip install --no-cache-dir -e git+https://github.com/ViderumGlobal/ckanext-archiver.git@v1.0.6-ckan-2.7#egg=ckanext-archiver && \
    pip install --no-cache-dir -r https://raw.githubusercontent.com/ViderumGlobal/ckanext-archiver/v1.0.6-ckan-2.7/requirements.txt && \
    # Create and update CKAN config
    paster --plugin=ckan make-config ckan ${APP_DIR}/production.ini && \
    paster --plugin=ckan config-tool ${APP_DIR}/production.ini "ckan.plugins = ${CKAN__PLUGINS}"

# Create a local user and group to run the app
RUN addgroup -g 92 -S ckan && \
    adduser -u 92 -h /srv/app -H -D -S -G ckan ckan

RUN touch /var/log/cron.log && \
    chmod 0666 /var/log/cron.log && \
    touch crontab.tmp && \
    echo '0 * * * * paster --plugin=ckanext-harvest harvester run -c /srv/app/production.ini >> /var/log/cron.log 2>&1' > crontab.tmp && \
    #echo '30 * * * * paster --plugin=ckanext-archiver archiver update -c /srv/app/production.ini >> /var/log/cron.log 2>&1' >> crontab.tmp && \
    crontab -u ckan crontab.tmp && \
    rm -rf crontab.tmp

COPY setup/cron.conf /etc/supervisord.d/cron.conf
#RUN ln -sf /proc/1/fd/1 /var/log/cron.log

COPY setup ${APP_DIR}
COPY setup/harvest.conf /etc/supervisord.d/harvest.conf
COPY setup/prerun_harvest.py ${APP_DIR}/prerun_harvest.py
COPY setup/redis.conf /etc/supervisord.d/redis.conf
COPY setup/uwsgi.conf /srv/app/uwsgi.conf
COPY setup/celery.conf /etc/supervisord.d/celery.conf
COPY setup/varnish/default.vcl /etc/varnish/default.vcl
RUN chown -R ckan:ckan /etc/varnish /var/lib/varnish

RUN chown ckan -R /srv/app

EXPOSE 5000

HEALTHCHECK --interval=10s --timeout=5s --retries=5 CMD curl --fail http://localhost:5000/api/3/action/status_show || exit 1

CMD ["/srv/app/start_ckan.sh"]
