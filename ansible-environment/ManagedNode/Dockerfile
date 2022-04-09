FROM ubuntu:18.04

RUN apt-get update \
    && apt-get install -y openssh-server curl python3 python3-pip net-tools nano less vim\
    && apt-get clean all \
    && pip3 install --upgrade pip virtualenv requests \
    && ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime \
    && mkdir /var/run/sshd \
    && mkdir /root/.ssh \
    && sed -ri 's/^PermitRootLogin\s+.*/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config \
    && mkdir /opt/ansible

COPY --chown=root id_rsa.pub /root/.ssh/authorized_keys

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
