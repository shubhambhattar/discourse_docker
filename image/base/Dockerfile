# NAME:     discourse/base
# VERSION:  release
FROM ubuntu:16.04

ENV PG_MAJOR 9.5 \
    RUBY_ALLOCATOR /usr/lib/libjemalloc.so.1 \
    COMPRESS_BROTLI 1

#LABEL maintainer="Sam Saffron \"https://twitter.com/samsaffron\""

RUN echo 2.0.`date +%Y%m%d` > /VERSION

RUN apt-get update && \
    apt-get install -y \
    lsb-release \
    sudo \
    curl

RUN echo "debconf debconf/frontend select Teletype" | debconf-set-selections && \
    echo "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) main restricted universe" > /etc/apt/sources.list && \
    echo "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc)-updates main restricted universe" >> /etc/apt/sources.list && \
    echo "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc)-security main restricted universe" >> /etc/apt/sources.list
    
RUN apt-get update && \
    apt-get -y install fping && \
    sh -c "fping proxy && echo 'Acquire { Retries \"0\"; HTTP { Proxy \"http://proxy:3128\";}; };' > /etc/apt/apt.conf.d/40proxy && apt-get update || true" && \
    apt-get -y install software-properties-common && \
    apt-mark hold initscripts && \
    apt-get -y upgrade && \
    curl http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | apt-key add - && \
    echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -sc)-pgdg main" | \
        tee /etc/apt/sources.list.d/postgres.list && \
    curl --silent --location https://deb.nodesource.com/setup_8.x | sudo bash - && \
    apt-get -y update
    
RUN apt-get -y install build-essential git wget \
                       libxslt-dev libcurl4-openssl-dev \
                       libssl-dev libyaml-dev libtool \
                       libxml2-dev gawk parallel \
                       postgresql-${PG_MAJOR} postgresql-client-${PG_MAJOR} \
                       postgresql-contrib-${PG_MAJOR} libpq-dev libreadline-dev \
                       language-pack-en cron anacron \
                       psmisc rsyslog vim whois brotli libunwind-dev

RUN sed -i -e 's/start -q anacron/anacron -s/' /etc/cron.d/anacron && \
    sed -i.bak 's/$ModLoad imklog/#$ModLoad imklog/' /etc/rsyslog.conf

RUN dpkg-divert --local --rename --add /sbin/initctl && \
    sh -c "test -f /sbin/initctl || ln -s /bin/true /sbin/initctl" && \
    apt-get -y install redis-server haproxy openssh-server
    
RUN cd / &&\
    apt-get -y install runit monit socat &&\
    mkdir -p /etc/runit/1.d &&\
    apt-get clean &&\
    rm -f /etc/apt/apt.conf.d/40proxy &&\
    locale-gen en_US &&\
    apt-get install -y nodejs &&\
    npm install -g uglify-js@"<3" &&\
    npm install svgo -g

COPY install-nginx /tmp/install-nginx
RUN /tmp/install-nginx

RUN apt-get -y install advancecomp jhead jpegoptim libjpeg-turbo-progs optipng

RUN mkdir /jemalloc-stable && cd /jemalloc-stable &&\
      wget https://github.com/jemalloc/jemalloc/releases/download/3.6.0/jemalloc-3.6.0.tar.bz2 &&\
      tar -xjf jemalloc-3.6.0.tar.bz2 && cd jemalloc-3.6.0 && ./configure --prefix=/usr && make && make install &&\
      cd / && rm -rf /jemalloc-stable

RUN mkdir /jemalloc-new && cd /jemalloc-new &&\
      wget https://github.com/jemalloc/jemalloc/releases/download/5.0.1/jemalloc-5.0.1.tar.bz2 &&\
      tar -xjf jemalloc-5.0.1.tar.bz2 && cd jemalloc-5.0.1 && ./configure --prefix=/usr --with-install-suffix=5.0.1 && make build_lib && make install_lib &&\
      cd / && rm -rf /jemalloc-new

RUN mkdir /gperftools && git clone https://github.com/gperftools/gperftools.git /gperftools &&\
      cd /gperftools && git checkout gperftools-2.5 && ./autogen.sh && ./configure --prefix=$PWD/lib &&\
      make install && mv lib/lib/libtcmalloc_minimal.so.4.3.0 /usr/lib && cd / && rm -rf /gperftools

RUN mkdir /lockless && cd /lockless && curl -O https://locklessinc.com/downloads/lockless_allocator_linux.tgz &&\
      tar zxf lockless_allocator_linux.tgz && mv lockless_allocator_linux/64bit/libllalloc.so.1.4 /usr/lib &&\
      cd / && rm -rf /lockless

RUN echo 'gem: --no-document' >> /usr/local/etc/gemrc &&\
    mkdir /src && cd /src && git clone https://github.com/sstephenson/ruby-build.git &&\
    cd /src/ruby-build && ./install.sh &&\
    cd / && rm -rf /src/ruby-build && (ruby-build 2.4.2 /usr/local)

RUN gem update --system

RUN gem install bundler --force &&\
    rm -rf /usr/local/share/ri/2.4.2/system &&\
    cd / && git clone https://github.com/SamSaffron/pups.git

COPY install-imagemagick /tmp/install-imagemagick
RUN /tmp/install-imagemagick

# Validate install
RUN ruby -Eutf-8 -e "v = \`convert -version\`; %w{png tiff jpeg freetype}.each { |f| unless v.include?(f); STDERR.puts('no ' + f +  ' support in imagemagick'); exit(-1); end }"

COPY install-pngcrush /tmp/install-pngcrush
RUN /tmp/install-pngcrush

COPY install-gifsicle /tmp/install-gifsicle
RUN /tmp/install-gifsicle

COPY install-pngquant /tmp/install-pngquant
RUN /tmp/install-pngquant

# clean up for docker squash
RUN rm -fr \
    /usr/share/man \
    /usr/share/doc \
    /usr/share/vim/vim74/tutor \
    /usr/share/vim/vim74/doc \
    /usr/share/vim/vim74/lang \
    /usr/local/share/doc \
    /usr/local/share/ruby-build \
    /root/.gem \
    /root/.npm \
    /tmp/* \
    /usr/share/vim/vim74/spell/en*


# this can probably be done, but I worry that people changing PG locales will have issues
# cd /usr/share/locale && rm -fr `ls -d */ | grep -v en`

RUN mkdir -p /etc/runit/3.d

COPY runit-1 /etc/runit/1
COPY runit-1.d-cleanup-pids /etc/runit/1.d/cleanup-pids
COPY runit-1.d-anacron /etc/runit/1.d/anacron
COPY runit-1.d-00-fix-var-logs /etc/runit/1.d/00-fix-var-logs
COPY runit-2 /etc/runit/2
COPY runit-3 /etc/runit/3
COPY boot /sbin/boot

COPY cron /etc/service/cron/run
COPY rsyslog /etc/service/rsyslog/run
COPY cron.d_anacron /etc/cron.d/anacron


# Discourse specific bits
RUN useradd discourse -s /bin/bash -m -U &&\
    mkdir -p /var/www &&\
    cd /var/www &&\
    git clone https://github.com/discourse/discourse.git &&\
    cd discourse &&\
    git remote set-branches --add origin tests-passed &&\
    chown -R discourse:discourse /var/www/discourse &&\
    cd /var/www/discourse &&\
    sudo -u discourse bundle install --deployment \
         --without test --without development &&\
    find /var/www/discourse/vendor/bundle -name tmp -type d -exec rm -rf {} +
