FROM nginx:1.10.2

MAINTAINER Scott Coulton

RUN useradd hacker 

ADD foo /tmp/foo

RUN chmod 0404 /tmp/foo 

USER hacker

WORKDIR /tmp

ADD dirtyc0w dirtyc0w

ADD cowroot cowroot

USER root

CMD ["nginx", "-g", "daemon off;"]



