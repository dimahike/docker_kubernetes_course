### 1. Create a new network
```bash
docker network create goals-net
```

### 2. Run the MongoDB container
```bash
docker run --name goals_db -d --rm --network goals-net -v data:/data/db -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=secret mongo
```
- `--name goals_db` - the name of the container
- `-d` - run in detached mode
- `--rm` - remove the container when it stops
- `--network goals-net` - connect to the network
- `-v data:/data/db` -  bind the store data to the volume. We store the data in the volume so that it persists even after the container removed.
- `-e MONGO_INITDB_ROOT_USERNAME=admin` - set the username for the root user
- `-e MONGO_INITDB_ROOT_PASSWORD=secret` - set the password for the root user
- `mongo` - the image to run

### 3. Run the backend container

3.1. Move to the backend folder

```bash
cd backend
```

3.2.  Add `Dockerfile` to the backend folder

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 80

ENV MONGO_DB_USERNAME=root
ENV MONGO_DB_PASSWORD=secret

CMD ["node", "app.js"]
```

- Add default environment variables to the Dockerfile
    ```dockerfile
    ...
    ENV MONGO_DB_USERNAME=root
    ENV MONGO_DB_PASSWORD=secret
    ...
    ```

3.3. Change mongo connect url  
```js
mongoose.connect(
	`mongodb://${process.env.MONGO_DB_USERNAME}:${process.env.MONGO_DB_PASSWORD}@goals_db:27017/course-goals?authSource=admin`,
);
```
- `process.env.MONGO_DB_USERNAME` - the username of the MongoDB
- `process.env.MONGO_DB_PASSWORD` - the password of the MongoDB
- `goals_db` - the name of the MongoDB container
- `27017` - the default port of MongoDB
- `course-goals` - the name of the database
- `?authSource=admin` - the query parameter to specify the authentication source

3.4. Build the image

```bash
docker build -t goals_backend .
```
- `-t goals_backend` - tag the image with the name `goals_backend`
- `.` - the path to the Dockerfile

3.5. Run the container
```bash
docker run --name goals_backend_app -d --rm -p 80:80 --network goals-net -v goals_logs:/app/logs -e MONGO_DB_USERNAME=admin -e MONGO_DB_PASSWORD=secret goals_backend
```
- `--name goals_backend_app` - the name of the container
- `-d` - run in detached mode
- `--rm` - remove the container when it stops
- `-p 80:80` - map the port 80 of the container to the port 80 of the host machine
- `--network goals-net` - connect to the network
- `-v goals_logs:/app/logs` - bind the logs folder to the volume
- `-e MONGO_DB_USERNAME=admin` - set the username for the MongoDB
- `-e MONGO_DB_PASSWORD=secret` - set the password for the MongoDB
- `goals_backend` - the image to run


#### Live source code update with nodemon
3.4. Add nodemon to app
```package.json
"scripts": {
   "start": "nodemon app.js"
 },
 devDependencies": {
   "nodemon": "^2.0.7"
 }
```

3.5. Change the `CMD` instruction in the Dockerfile
```dockerfile
CMD ["npm", "start"]
```

3.6. Build the image
```bash
docker build -t goals_backend:nodemon .
```
3.6. Run the container
```bash
docker run --name goals_backend_app -d  --rm -p 80:80 --network goals-net -v goals_logs:/app/logs -v "$(pwd):/app:ro" -v /app/node_modules -e MONGO_DB_USERNAME=admin -e MONGO_DB_PASSWORD=secret   goals_backend:nodemon
````
- `-v "$(pwd):/app:ro"` - bind the current directory to the `/app` folder in the container in read-only mode. We can use the absolute path instead of `$(pwd)`(Mac OS). 
- `-v /app/node_modules` - bind the `node_modules` folder in the container to the temporary volume
- `goals_backend:nodemon` - the image to run

### 4. Run the frontend container

4.1. Move to the frontend folder

```bash
cd frontend
```

4.2. Add `Dockerfile` to the frontend folder
```bash
docker build -t goals_frontend .
```
- `-t goals_frontend` - tag the image with the name `goals_frontend`
- `.` - the path to the Dockerfile

4.3. Build the image
```bash
docker run --name goals_frontend_app -d --rm  -p 3000:3000 --network goals-net goals_frontend
```
- `--name goals_frontend_app` - the name of the container
- `-d` - run in detached mode
- `--rm` - remove the container when it stops
- `-p 3000:3000` - map the port 3000 of the container to the port 3000 of the host machine
- `--network goals-net` - connect to the network
- `goals_frontend` - the image to run

#### Live source code update with react-scripts

4.3. Add react-scripts to the frontend
```bash
docker run --name goals_frontend_app -d --rm -p 3000:3000 --network goals-net -v "$(pwd)/src:/app/src:ro" -v /app/node_modules goals_frontend
```
- `-v "$(pwd)/src:/app/src:ro"` - bind the current directory to the `/app` folder in the container in read-only mode. We can use the absolute path instead of `$(pwd)`(Mac OS).
- `-v /app/node_modules` - bind the `node_modules` folder in the container to the temporary volume
- `goals_frontend` - the image to run

---
#### Notes:

- Recommend to add the `.dockerignore` file to the backend and frontend folders to exclude unnecessary files and folders from the context.
    ```.dockerignore
    node_modules
    package-lock.json
    Dockerfile
    .env
    .git
    .DS_Store
    .idea
    ```