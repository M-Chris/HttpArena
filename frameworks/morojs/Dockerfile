FROM node:26-trixie-slim AS build
WORKDIR /app
COPY package.json .
# @morojs/engine ships prebuilt binaries, so no toolchain is needed here
RUN npm install --omit=dev --no-audit --no-fund

FROM node:26-trixie-slim
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY app.mjs .
COPY views ./views
ENV NODE_ENV=production
EXPOSE 8080
CMD ["node", "app.mjs"]
