
FROM node:18-alpine

WORKDIR /app

# Install tools required to compile native dependencies
RUN apk add --no-cache python3 make g++

COPY package*.json ./

RUN npm install --omit=dev

COPY . .

ENV NODE_ENV=production

EXPOSE 3001

CMD ["npm", "run", "start"]

