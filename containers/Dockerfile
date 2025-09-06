FROM docker.io/library/alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /
RUN git -c advice.detachedHead=false clone --branch $TAG --depth=1 --recurse-submodules https://github.com/pfm-powerforme/dagu.git source
WORKDIR /source/


FROM docker.io/library/node:lts-alpine AS frontend
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN apk update && apk add --no-cache \
	gcc musl-dev g++ make py3-pip
RUN corepack enable
WORKDIR /
COPY --from=git /source/ui/ /frontend/
WORKDIR /frontend/
RUN rm -rf node_modules; \
  pnpm install --frozen-lockfile; \
  pnpm build


FROM docker.io/library/golang:alpine AS backend
ARG TAG
ENV TAG=${TAG:-v0.0.1} \
    GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux
RUN apk update && apk add --no-cache \
	git
WORKDIR /
COPY --from=git /source/ backend/
WORKDIR /backend/
RUN go mod download
COPY --from=frontend /frontend/dist/ internal/frontend/assets/
RUN go build -ldflags="-X 'main.version=$(echo "${TAG}" | sed 's/^v//')-$(git rev-parse --short HEAD)'" -o dagu ./cmd


FROM docker.io/library/alpine:latest AS runtime
ARG USER="dagu"
ARG USER_UID=1000
ARG USER_GID=$USER_UID
ARG DAGU_HOME="/var/lib/dagu"
RUN apk update && apk add --no-cache \
    jq tzdata sudo shadow bash \
    curl wget gcc musl-dev python3 py3-pip
COPY --from=backend /backend/dagu /usr/local/bin/dagu
COPY --from=git /source/entrypoint.sh /entrypoint.sh
RUN set -eux; \
    (groupadd -g "${USER_GID}" "${USER}" || groupadd "${USER}") && \
    (useradd -m -d /dagu \
              -u "${USER_UID}" \
              -g "$(getent group "${USER}" | cut -d: -f3)" \
              -s /bin/bash \
              "${USER}" \
    || useradd -m -d /dagu \
               -g "$(getent group "${USER}" | cut -d: -f3)" \
               -s /bin/bash \
               "${USER}") && \
    chmod +x /entrypoint.sh
RUN echo "dagu ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/99-dagu \
    && chmod 440 /etc/sudoers.d/99-dagu
RUN mkdir -p "${DAGU_HOME}" && \
    chown -R "${USER_UID}:${USER_GID}" "${DAGU_HOME}" && \
    chmod 755 "${DAGU_HOME}"
WORKDIR /home/dagu
ENV DAGU_HOST=0.0.0.0
ENV DAGU_PORT=8080
ENV DAGU_HOME=$DAGU_HOME
ENV DAGU_TZ="Asia/Shanghai"
ENV PUID=${USER_UID}
ENV PGID=${USER_GID}
ENV DOCKER_GID=-1
EXPOSE 8080
ENTRYPOINT ["/entrypoint.sh"]
CMD ["dagu", "start-all"]
