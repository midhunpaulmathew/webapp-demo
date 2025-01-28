FROM nginx:latest

WORKDIR /usr/share/nginx/html
COPY ./static-site /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
