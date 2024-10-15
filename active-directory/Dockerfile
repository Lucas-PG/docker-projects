FROM ubuntu:20.04


ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y samba krb5-user smbclient openssl

RUN mkdir -p /etc/samba /var/lib/samba/private

COPY smb.conf /etc/samba/smb.conf

EXPOSE 389 636
CMD ["samba", "--no-process-group", "--foreground"]
