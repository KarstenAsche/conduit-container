FROM node:20 AS build

WORKDIR /app

# Install dependencies using the existing lockfile.
COPY package*.json ./
RUN npm ci

# Copy the application and build it for deployment.
COPY . .
RUN npm run build

# Stage 2: Serve the compiled application with Nginx.
FROM nginx:alpine

# Copy the Angular build output into the Nginx web directory.
COPY --from=build /app/dist/angular-conduit/ /usr/share/nginx/html/

# Configure Angular routing and the backend reverse proxy.
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80