# syntax=docker/dockerfile:1

# ---------- Stage 1: build the static site ----------
FROM node:22-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# VITE_API_URL is baked into the JS bundle at build time (see src/api/axios.js).
# It must be the public origin users reach nginx on, e.g. http://your-server-ip
# or https://ims.example.com. Changing it requires rebuilding this image.
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# ---------- Stage 2: nginx serves the build and proxies /api ----------
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
