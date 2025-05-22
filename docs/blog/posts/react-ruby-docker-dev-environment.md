---
date: 2024-05-08
categories:
  - react
  - ruby
  - docker
  - devops
---
# React and Ruby Docker Development Environment

## Background

Lately I am working on a project to dockerize an e2e application with React as the front end and Ruby on Rails as the backend. I would like to set up a docker-compose file so that:

1. user can bring up the e2e environment with a single command
1. the environment supports hot reload of code for both the React and Ruby on Rails parts
1. it can be served as a testing environment for e2e test
1. developers can run manual test in a local environment

<!-- more -->

## React Dockerfile

``` bash
FROM node:lts-alpine3.17

WORKDIR /app

COPY package.json /app/
COPY yarn.lock /app/
RUN yarn install

COPY . /app

RUN yarn run build
CMD yarn start
```

## docker-compose

``` bash
version: '3'
services:
  api:
    build: ./backend
    command: bundle exec rails server -p 3000 -b '0.0.0.0'
    ports:
      - "3000:3000"
    volumes:
      - "./backend/:/rails"
  web:
    build: ./frontend
    environment:
      REACT_APP_BACKEND_URL: "http://localhost:3000"
      WATCHPACK_POLLING: "true"
    command: npm start
    ports:
      - "8080:3000"
    volumes:
      - ./frontend/:/app
    depends_on:
      - api
```

note that the followings are essential to enable hot reload for both React and Rails:

### React Hot Reload
``` bash
  web:
    ...
    environment:
      WATCHPACK_POLLING: "true"
    command: npm start
    ...
    volumes:
      - ./frontend/:/app
```
### Ruby on Rails Hot Reload
``` bash 
  api:
    ...
    command: bundle exec rails server -p 3000 -b '0.0.0.0'
    ...
    volumes:
      - "./backend/:/rails"
```

## Connecting Front End to Back End

When the React application was trying to consume the Rails API, CORS error was reported. The following is the fix to make them communicate properly:

### backend/Gemfile

``` bash
gem "rack-cors"
```

### backend/config/initializers/cors.rb

``` bash
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'localhost:3000',
            '127.0.0.1:3000',
            'localhost:8080',
            '127.0.0.1:8080'

    resource '*',
             headers: :any,
             methods: %i[get post put patch delete options head]
  end
end
Rails.application.config.hosts += ['localhost:3000',
                                   'localhost',
                                   '127.0.0.1:3000',
                                   'localhost:8080',
                                   '127.0.0.1:8080'
                                ]
```

## Sample React Component to Consume Ruby on Rails API

``` js
import React, { useState, useEffect }  from 'react';
const BackendTest = () => {
const [error, setError] = useState(null);
    const [isLoaded, setIsLoaded] = useState(false);
    const [foo, setFoo] = useState([]);
    useEffect(() => {
        fetch(process.env.REACT_APP_BACKEND_URL + "/v0/health-check")
            .then(res => res.json())
            .then(
                (data) => {
                    setIsLoaded(true);
                    setFoo(data);
                },
                (error) => {
                    setIsLoaded(true);
                    setError(error);
                }
            )
      }, [])
if (error) {
        return <div>Error: {error.message}</div>;
    } else if (!isLoaded) {
        return <div>Loading...</div>;
    } else {
        console.log(foo)
        return (
            <div>{foo.data}</div>
        );
    }
}
export default BackendTest;
```
