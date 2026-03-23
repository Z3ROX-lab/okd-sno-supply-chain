FROM harbor.okd.lab/library/httpd:alpine
LABEL org.opencontainers.image.source="https://github.com/Z3ROX-lab/okd-sno-supply-chain"
LABEL org.opencontainers.image.description="Supply Chain Security Demo App"
# Fix CVE-2026-32767 - libexpat upgrade
RUN apk upgrade --no-cache libexpat
COPY src/ /usr/local/apache2/htdocs/
EXPOSE 80
