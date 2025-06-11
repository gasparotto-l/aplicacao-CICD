# Etapa de build com Node.js
FROM node:18-alpine AS build

# Diretório de trabalho
WORKDIR /app

# Copia arquivos de dependências
COPY package.json package-lock.json ./

# Instala as dependências com cache otimizado
RUN npm ci

# Copia o restante dos arquivos para build
COPY . .

# Build da aplicação React/Vite/etc.
RUN npm run build

# Etapa final: imagem mínima com Nginx
FROM nginx:alpine

# Copia os arquivos estáticos gerados no build
COPY --from=build /app/build /usr/share/nginx/html

# Substitui configuração padrão do Nginx para rodar na porta 3000
RUN rm /etc/nginx/conf.d/default.conf && \
    echo 'server { \
        listen 3000; \
        server_name localhost; \
        location / { \
            root /usr/share/nginx/html; \
            index index.html index.htm; \
            try_files $uri $uri/ /index.html; \
        } \
    }' > /etc/nginx/conf.d/default.conf

# Expõe a porta do servidor
EXPOSE 3000

# Comando padrão do Nginx
CMD ["nginx", "-g", "daemon off;"]
