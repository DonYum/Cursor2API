FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production
ENV CURSOR_SDK_BRIDGE_HOST=0.0.0.0
ENV CURSOR_SDK_BRIDGE_PORT=8792

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY scripts/cursor-sdk-local-agent-bridge.mjs ./scripts/cursor-sdk-local-agent-bridge.mjs

EXPOSE 8792

USER node

CMD ["node", "scripts/cursor-sdk-local-agent-bridge.mjs"]
