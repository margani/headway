FROM node:16-slim as builder

RUN yarn global add @quasar/cli

COPY frontend /frontend

WORKDIR /frontend

RUN yarn install \
  && quasar build

FROM nginx

COPY --from=builder /frontend/dist/spa/ /usr/share/nginx/html/

COPY nginx.conf.template /etc/nginx/templates/nginx.conf.template

COPY bbox.txt /usr/share/nginx/html/bbox.txt

ENV HEADWAY_PUBLIC_URL=http://127.0.0.1:8080
ENV HEADWAY_HTTP_PORT=8080
ENV HEADWAY_RESOLVER=127.0.0.11
ENV HEADWAY_GRAPHHOPPER_URL=http://graphhopper:8989
ENV HEADWAY_PHOTON_URL=http://photon:2322
ENV HEADWAY_TILESERVER_URL=http://mbtileserver:8000
ENV HEADWAY_NOMINATIM_URL=http://nominatim:8080
# for escaping $ in nginx template
ENV ESC=$
ENV NGINX_ENVSUBST_OUTPUT_DIR=/etc/nginx
