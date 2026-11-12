# Ejecuta la build de Astro (carpeta dist/) con Node.
# El servidor de Astro necesita sus dependencias de producción: se instalan aquí,
# dentro de la imagen (no se copia node_modules del equipo).
FROM node:22-alpine

WORKDIR /app
ENV NODE_ENV=production \
    HOST=0.0.0.0 \
    PORT=4321

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY dist ./dist

EXPOSE 4321
CMD ["node", "./dist/server/entry.mjs"]
