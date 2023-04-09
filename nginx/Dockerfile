#FROM debian:latest
FROM nginx

# Install nginx and adjust nginx config to stay in foreground
#COPY nginx.conf /etc/nginx/nginx.conf
#RUN apt-get update && apt-get install --no-install-recommends -y nginx; echo "daemon off;" >> /etc/nginx/nginx.conf

COPY nginx.conf /etc/nginx/nginx.conf
RUN ln -sf /dev/stdout /var/log/nginx/access_main.log && ln -sf /dev/stdout /var/log/nginx/access_json.log

# Expose HTTP
#EXPOSE 80

# Start nginx
#CMD ["/usr/sbin/nginx"]
