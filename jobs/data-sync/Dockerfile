FROM python:3.12.2-bullseye AS development_build

USER root

ARG VCS_REF="missing"
ARG BUILD_DATE="missing"

ENV VCS_REF=${VCS_REF}
ENV BUILD_DATE=${BUILD_DATE}
ENV PORT=8080

LABEL org.label-schema.vcs-ref=${VCS_REF} \
    org.label-schema.build-date=${BUILD_DATE}

LABEL maintainer="thorwolpert"
LABEL vendor="BCROS"

ARG APP_ENV \
  # Needed for fixing permissions of files created by Docker:
  UID=1000 \
  GID=1000

ENV APP_ENV=${APP_ENV} \
  # python:
  PYTHONFAULTHANDLER=1 \
  PYTHONUNBUFFERED=1 \
  PYTHONHASHSEED=random \
  PYTHONDONTWRITEBYTECODE=1 \
  # pip:
  PIP_NO_CACHE_DIR=1 \
  PIP_DISABLE_PIP_VERSION_CHECK=1 \
  PIP_DEFAULT_TIMEOUT=100 \
  PIP_ROOT_USER_ACTION=ignore \
  # poetry:
  POETRY_VERSION=1.8.2 \
  POETRY_NO_INTERACTION=1 \
  POETRY_VIRTUALENVS_CREATE=false \
  POETRY_CACHE_DIR='/var/cache/pypoetry' \
  POETRY_HOME='/usr/local'

SHELL ["/bin/bash", "-eo", "pipefail", "-c"]

RUN apt-get update && apt-get upgrade -y \
  && apt-get install --no-install-recommends -y \
    bash \
    brotli \
    build-essential \
    curl \
    gettext \
    git \
    libpq-dev \
    wait-for-it \
  && curl -sSL 'https://install.python-poetry.org' | python - \
  && poetry --version \
  # Cleaning cache:
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && apt-get clean -y && rm -rf /var/lib/apt/lists/*

WORKDIR /code

RUN groupadd -g "${GID}" -r web \
  && useradd -d '/code' -g web -l -r -u "${UID}" web \
  && chown web:web -R '/code'

# Copy only requirements, to cache them in docker layer
COPY --chown=web:web ./poetry.lock ./pyproject.toml /code/

# Copy the run.sh script from jobs/ar-reminder/ to the /code directory
# Ensure run.sh has execute permissions
COPY --chown=web:web ./run.sh /code/run.sh
RUN chmod 755 /code/run.sh

# Project initialization:
RUN --mount=type=cache,target="$POETRY_CACHE_DIR" \
  echo "$APP_ENV" \
  && poetry version \
  # Install deps:
  && poetry run pip install -U pip \
  && poetry install \
    $(if [ -z ${APP_ENV+x} ] | [ "$APP_ENV" = 'production' ]; then echo '--only main'; fi) \
    --no-interaction --no-ansi

COPY --chown=web:web ./src /code/src
COPY --chown=web:web ./README.md /code
COPY --chown=web:web ./logging.conf /code

RUN chmod -R 755 /code

USER web

# The following stage is only for production:
FROM development_build AS production_build

ENV PYTHONPATH=/code/src

CMD [ "/bin/sh", "/code/run.sh" ] 