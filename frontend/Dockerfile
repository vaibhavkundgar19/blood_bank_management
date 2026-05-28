# ------------------------------------------------------------------------------
# 🏗️ Stage 1: Build React/Vite App
# ------------------------------------------------------------------------------

FROM node:21 AS frontend-builder

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

ARG VITE_API_PATH

ENV NODE_ENV=production
ENV VITE_API_PATH=${VITE_API_PATH}

RUN npm run build

# ------------------------------------------------------------------------------
# 🚀 Stage 2: Serve Frontend Using Nginx
# ------------------------------------------------------------------------------

FROM nginx:alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy build files from builder stage
COPY --from=frontend-builder /app/dist /usr/share/nginx/html

# Expose nginx port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]