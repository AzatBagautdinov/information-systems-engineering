# Используем официальный Node.js образ
FROM node:18-alpine AS development

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем package.json и package-lock.json
COPY package*.json ./

# Устанавливаем зависимости
RUN npm install

# Копируем исходный код
COPY . .

# Стадия для тестирования
FROM development AS test
RUN npm test

# Продакшн стадия
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY --from=development /app/src ./src

# Создаём пользователя для безопасности
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

USER nextjs

# Открываем порт
EXPOSE 3000

# Команда по умолчанию
CMD ["npm", "start"]
