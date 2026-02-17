FROM node:20

WORKDIR /app

COPY . .

RUN npm install
RUN npm run build

EXPOSE 80

CMD ["npm", "run", "start", "--", "-p", "80"]
