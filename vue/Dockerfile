# first stage builds vue
FROM node:18 as build-stage
WORKDIR /build
COPY start-container /usr/local/bin/start-container
RUN chmod +x /usr/local/bin/start-container
# COPY . .
# RUN npm install
# RUN bash /build/start-container
# RUN npm run build

# second stage copies the static dist files and Node server files
# FROM node:18 as production-stage
# WORKDIR /app
# COPY ./config ./config/
# COPY package.json vueBaseAppServer.js ./
# COPY --from=build-stage /build/dist/ dist/
# RUN npm install --omit=dev
# RUN rm -rf build

# open port 3000 and run Node server
EXPOSE 3000
# CMD ["npm", "install"]
# CMD ["npm", "run", "build"]
# CMD ["node", "vueBaseAppServer.js"]
ENTRYPOINT ["start-container"]