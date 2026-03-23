FROM harbor.okd.lab/library/httpd:alpine
LABEL org.opencontainers.image.source="https://github.com/Z3ROX-lab/okd-sno-supply-chain"
LABEL org.opencontainers.image.description="Supply Chain Security Demo App"
# Fix CVE-2026-32767 - libexpat upgrade via proxy
RUN echo "https_proxy=http://10.128.0.2:8888" >> /etc/environment && \
    http_proxy=http://10.128.0.2:8888 \
    https_proxy=http://10.128.0.2:8888 \
    apk upgrade --no-cache libexpat
COPY src/ /usr/local/apache2/htdocs/
EXPOSE 80
