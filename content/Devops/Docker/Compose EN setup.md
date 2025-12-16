## Create a simple application on Python and run it in Docker Compose

Create a new directory and create a directory structure:

```bash
mkdir softserve
cd softserve
mkdir server client
touch docker-compose.yml
```

In the **server** folder create the following files:

```bash
cd server
touch server.py index.html Dockerfile
```

Open server.py file in an editor and write the following code to it:

```
nano server.py
```

```javascript
#!/usr/bin/env python3
import http.server
import socketserver
handler = http.server.SimpleHTTPRequestHandler
with socketserver.TCPServer(("", 1234), handler) as httpd:
     httpd.serve_forever()
```

Save the file and exit (Ctrl+O, Ctrl+X)

### Shebang (#!/usr/bin/env python3):

This line is known as a shebang and specifies the interpreter that should be used to execute the script. In this case, it points to the Python 3 interpreter.

### Import modules:

import http.server import socketserver

These lines import the necessary modules for creating an HTTP server (http.server) and handling sockets (socketserver).

### Handler Definition:

handler = http.server.SimpleHTTPRequestHandler

This line creates an instance of the SimpleHTTPRequestHandler class from the http.server module. This handler is a basic HTTP request handler that serves files from the current directory.

### Server Configuration and Start:

with socketserver.TCPServer(("", 1234), handler) as httpd: httpd.serve_forever()

This block of code is a context manager that creates a TCP server on port 1234 using the specified handler. The handler is a class that defines how the server will handle incoming requests.

The serve_forever() method tells the server to start listening for requests and handling them until it is stopped.

Open the index.html file in the editor for editing:

```
nano index.html
```

Paste the following code:

```
My first docker-compose task!
```

Save the file and exit.

Open the Dockerfile file in the editor for editing:

```
nano Dockerfile
```

And copy the following code that creates a docker container for our application.

```dockerfile
FROM python:latest
ADD server.py /server/
ADD index.html /server/
WORKDIR /server/
```

Save the file and exit

**Explanation:**

### FROM python:latest:

This line specifies the base image for the new image. In this case, it uses the official Python image from Docker Hub, tagged as "latest." This means the image will be built on top of the latest version of the Python image available at the time of the build.

### ADD server.py /server/:

This line adds the local file server.py to the image at the /server/ directory. The ADD command is used to copy files or directories from the build context (the directory containing the Dockerfile and any referenced files) to the image. In this case, it copies the Python server script server.py to the /server/ directory inside the image.

### ADD index.html /server/:

Similar to the previous line, this command adds the local file index.html to the image at the /server/ directory. It copies the HTML file to the same directory inside the image.

### WORKDIR /server/:

Sets the working directory for any subsequent instructions in the Dockerfile to /server/. The WORKDIR command is used to change the current working directory within the image. In this case, it ensures that any future commands will be executed in the context of the /server/ directory.

## Create a client

Go to client folder:

```javascript
cd ~/softserve/client
```

Create following files:

```
touch Dockerfile client.py
```

Open and edit **client.py** in editor:

```
nano client.py
```

Insert following code to file:

```python
#!/usr/bin/env python3
import urllib.request
fp = urllib.request.urlopen("http://localhost:1234/")
encodedContent = fp.read()
decodedContent = encodedContent.decode("utf8")
print(decodedContent)
fp.close()
```

Save the file and exit the editor.

This Python script is a simple HTTP client that retrieves the content of a webpage located at "http://localhost:1234/" and prints it to the console. Let's break down each part of the code:

### Import module:

import urllib.request

This line imports the urllib.request module, which provides an interface for opening and reading URLs.

### HTTP Request:

fp = urllib.request.urlopen("http://localhost:1234/")

This line opens a connection to the URL "http://localhost:1234/" using the urlopen function from urllib.request. The result (fp) is a file-like object representing the content of the URL.

### Read Content:

encodedContent = fp.read()

This line reads the content of the URL (in bytes) and stores it in the variable encodedContent.

### Decode Content:

decodedContent = encodedContent.decode("utf8")

This line decodes the content from bytes to a Unicode string using the UTF-8 encoding and stores it in the variable decodedContent.

### Print Content:

print(decodedContent)

This line prints the decoded content to the console.

### Close Connection:

fp.close()

This line closes the connection to the URL.

In summary, this script performs a simple HTTP GET request to "http://localhost:1234/", retrieves the content, decodes it, and prints it to the console. It's a basic example of making a web request and handling the response in Python.

Open **Dockerfile** in editor:

```
nano Dockerfile
```

Insert next code in to file:

```dockerfile
FROM python:latest
ADD client.py /client/
WORKDIR /client/
```

Save the file and exit. Fine. Now let's return to the root directory of the project and edit the docker-compose.yml file.

```bash
cd ..
nano docker-compose.yml
```

Insert following code:

```yaml
version: "3.3"
services:
   server:
       build: server/
       command: python ./server.py
       ports:
           - 1234:1234
   client:
       build: client/    
       command: python ./client.py
       network_mode: host
       depends_on:
            - server

```

Save the file and exit

This code represents a docker-compose.yml file, which is used to define and manage multi-container Docker applications. It specifies the configuration for two services: server and client. Let's break down each part of the code:

```javascript
version: "3.3"
```

This indicates the version of the Docker Compose file format being used. In this case, it's version 3.3.

```yaml
services:
   server:
       build: server/
       command: python ./server.py
       ports:
           - 1234:1234
```

**build: server/** : This specifies that the Docker image for the server service should be built from the server/ directory. The server/ directory likely contains a Dockerfile for building the server image.

**command: python ./server.py** : This sets the command to be executed when the server container starts. It runs the Python script server.py.

**ports: - 1234:1234** : This maps port 1234 from the host to port 1234 in the server container, allowing external access to the server.

```yaml
   client:
       build: client/
       command: python ./client.py
       network_mode: host
       depends_on:
            - server
```

**build: client/** : Similar to the server service, this specifies that the Docker image for the client service should be built from the client/ directory.

**command: python ./client.py** : Sets the command to be executed when the client container starts. It runs the Python script client.py.

**network_mode: host** : This allows the client container to use the host's network stack. It effectively runs the container in the host's network namespace, allowing it to connect to services running on the host directly.

**depends_on: - server** : Specifies that the client service depends on the server service. This ensures that the server container is started before the client container.

In summary, this docker-compose.yml file defines two services, server and client, with the server service building from the server/ directory and running a Python script, and the client service building from the client/ directory, running a different Python script, and depending on the server service. The client service is configured to use the host's network stack. This configuration is suitable for a scenario where a client and server application are interacting, and they need to communicate over the network.

### Create and run

Build project:

```
docker-compose build
```

Run services

```
docker-compose up
```

Check services in browser:

[Access to web page](https://00ffba03eedf-10-244-3-121-1234.spch.r.killercoda.com/)

Open a new terminal window (Tab2) and run following command:

```bash
cd softserve
```

```
docker-compose logs
```

Outputs the logs of containers for the specified services.

List containers

```
docker-compose ps
```

Down the service.

```
docker-compose down
```

Up the service:

```
docker-compose up
```

Stopinng the service

```
docker-compose stop
```

Starting the service

```
docker-compose start
```

Restarting the service

```
docker-compose  restart
```