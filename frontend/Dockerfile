# Etapa de build com Node.js
FROM node:18-alpine AS build

# Diretório de trabalho
WORKDIR /app

# Copia arquivos de dependências
COPY package.json package-lock.json ./

# Instala as dependências com cache otimizado
RUN npm ci


COPY . .


RUN npm run build


FROM nginx:alpine


RUN rm -rf /usr/share/nginx/html/*
RUN rm /etc/nginx/conf.d/default.conf


COPY --from=build /app/build /usr/share/nginx/html 2>/dev/null || \
COPY --from=build /app/dist /usr/share/nginx/html


RUN echo 'server { \
    listen 3000; \
    server_name localhost; \
    root /usr/share/nginx/html; \
    index index.html index.htm; \
    \
    # Configuração para SPA (Single Page Application) \
    location / { \
        try_files $uri $uri/ /index.html; \
    } \
    \
    # Cache para arquivos estáticos \
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ { \
        expires 1y; \
        add_header Cache-Control "public, no-transform"; \
    } \
    \
    # Configurações de segurança \
    add_header X-Frame-Options "SAMEORIGIN" always; \
    add_header X-Content-Type-Options "nosniff" always; \
    add_header X-XSS-Protection "1; mode=block" always; \
    \
    # Logs de acesso \
    access_log /var/log/nginx/access.log; \
    error_log /var/log/nginx/error.log; \
}' > /etc/nginx/conf.d/default.conf


EXPOSE 3000


CMD ["nginx", "-g", "daemon off;"]