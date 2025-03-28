## About
The BlueChatter app is a very simple chat/IRC type app for your browser.
It is very basic, you just go to the app, enter a user name and start
chatting.

For a demonstration of this application you can watch the following 
YouTube video.

[![BlueChatter](https://img.youtube.com/vi/i7_dQQy40ZQ/0.jpg?time=1398101975441)](http://youtu.be/i7_dQQy40ZQ)


## Technologies
BlueChatter uses [Node.js](http://nodejs.org/) and 
[Express](http://expressjs.com/) for the server.  On the frontend 
BlueChatter uses [Bootstrap](http://getbootstrap.com/) and 
[JQuery](http://jquery.com/).  The interesting part of this application 
is how the communication of messages is done.  The application uses [long 
polling](http://en.wikipedia.org/wiki/Push_technology#Long_polling) to enable 
the clients (browsers) to listen for new messages.  Once the
app loads a client issues a request to the server.  The server waits to respond
to the request until it receives a message.  If no message is received from any
of the chat participants it responds back to the client with a 204 - no content.
As soon as the client gets a response from the server, regardless of whether that
response contains a message or not, the client will issue another request and
the process continues.

One of the goals of this application is to demonstrate scaling in BlueMix.
As we know when you scale an application in BlueMix you essentially are
creating multiple instance of the same application which users will connect
to at random.  In other words there are multiple BlueChatter servers running 
at the same time.  So how do we communicate chat messages between the servers?
We use the [pubsub feature of Redis](http://redis.io/topics/pubsub) to solve 
this.  All servers bind to a single
Redis instance and each server is listening for messages on the same channel.
When one chat server receives a chat message it publishes an event to Redis
containing the message.  The other servers then get notifications of the new
messages and notify their clients of the.  This design allows BlueChatter to
scale nicely to meet the demand of its users.

## Deploying To BlueMix

The easiest way to deploy BlueChatter is to click the "Deploy to Bluemix"
button below.

[![Deploy to Bluemix](https://bluemix.net/deploy/button.png)](https://bluemix.net/deploy?repository=https://github.com/IBM-Bluemix/bluechatter)

### Using The Command Line
Make sure you have the Cloud Foundry Command Line installed and you
are logged in.

    $ cf login -a https://api.ng.bluemix.net

Next you need to create a Redis service for the app to use.  Lets use the RedisCloud service.

    $ cf create-service rediscloud 25mb redis-chatter

### Using The Cloud Foundry CLI

Now just push the app, we have a manifest.yml file so the command 
is very simple.
    
    $ git clone https://github.com/CodenameBlueMix/bluechatter.git	
	$ cd bluechatter
    $ cf push my-bluemix-chatter-app-name

### Using IBM DevOps Services (JazzHub)
If you would like you can also deploy the project to BlueMix using
IBM's DevOps Services (JazzHub).  You can find the project 
[here](https://hub.jazz.net/project/rjbaxter/bluechatter/overview) on
JazzHub.  To deploy you will need to login to JazzHub. Then click
Edit Code in the upper right hand corner.  In the editor click the 
Deploy button in the toolbar.


## Scaling The App

Since we are using Redis to send chat messages, you can scale this application
as much as you would like and people can be connecting to various servers
and still receive chat messages.  To scale you app you can run the following
command.

    $ cf scale my-blue-chatter-app-name -i 5

Then check your that all your instances are up and running.

    $ cf app my-blue-chatter-app-name

When you connect to app you can see which instance you are connecting to
in the footer of the application.  If you have more than one instance
running chances are the instance id will be different between two different
browsers.

### Docker

Bluechatter can be run inside a Docker container locally or in the 
IBM Containers Service in Bluemix.

#### Running In A Docker Container Locally

To run locally you must have Docker and Docker Compose installed locally.
If you are using OSX or Windows, both should be installed by default
when you install [Docker Toolbox](https://www.docker.com/toolbox).  For
Linux users follow the [instructions](https://docs.docker.com/compose/install/)
on the Docker site.

Once you have Docker and Docker Compose installed run the following commands
from the application root to start the Bluechatter application.

```
$ docker-compose build
$ docker-compose up
```
On OSX and Windows you will need the IP address of your Docker Machine VM to test the application.
Run the following command, replacing `machine-name` with the name of your Docker Machine.

```
$ docker-machine ip machine-name
```
Now that you have the IP go to your favorite browser and enter the IP in the address bar,
you should see the app come up.  (The app is running on port 80 in the container.)

On Linux you can just go to [http://localhost](http://localhost).

#### Running The Container On Bluemix

Before running the container on Bluemix you need to have gone through the steps to setup
the IBM Container service on Bluemix.  Please review the 
[documentation](https://www.ng.bluemix.net/docs/containers/container_index.html) on Bluemix before
continuing.

The following instruction assume you are using the [Cloud Foundry 
CLI IBM Containers Plugin](https://www.ng.bluemix.net/docs/containers/container_cli_cfic.html#container_cli_cfic_install).
If you are not using the plugin, execute the equivalent commands for your CLI solution.

Make sure you are logged into Bluemix from the Cloud Foundry CLI and the IBM Containers plugin.

```
$ cf login -a api.ng.bluemix.net
$ cf ic login
```

After you have logged in you can build an image for the Bluechatter application on Bluemix.
From the root of Bluechatter application run the following commnand replacing namespace
with your namespace for the IBM Containers service.  (If you don't know what your namespace is
run `$ cf ic namespace get`.)

```
$ cf ic build -t namespace/bluechatter ./
```

The above `build` command will push the code for Bluechatter to the IBM Containers Docker service
and run a `docker build` on that code.  Once the build finishes the resulting image will be 
deployed to your Docker registry on Bluemix.  You can verify this by running 

```
$ cf ic images
```

You should see a new image with the tag `namespace/bluechatter` listed in the images available to you.
You can also verify this by going to the [catalog](https://console.ng.bluemix.net/catalog/) on Bluemix,
in the containers section you should see the Bluechatter image listed.

Before you can start a container from the image we are going to need a Redis service for our container to use.
To do this in Bluemix we will need what is called a "bridge app".  Follow the [
instructions](https://www.ng.bluemix.net/docs/containers/container_binding_ov.html#container_binding_ui) on
Bluemix for how to create a bridge app from the UI.  Make sure you bind a 
[Redis Cloud](https://console.ng.bluemix.net/catalog/redis-cloud/) service to the bridge
app and name it "redis-chatter".

Once your bridge app is created follow the 
[instructions](https://www.ng.bluemix.net/docs/containers/container_single_ov.html#container_single_ui) 
on Bluemix for deploying a container based on the Bluechatter image.  Make sure you request a public IP
address, expose port 80, and bind to the bridge app you created earlier.  Once your container starts you
go to the public IP address assigned to the app in your browser and you should see the Bluechatter UI.

## Testing

After the app is deployed you can test it by opening two different browsers
and navigating to the URL of the app.  YOU MUST USE TWO DIFFERENT BROWSERS
AND NOT TWO TABS IN THE SAME BROWSER.  Since we are using long polling
browsers will not actually make the same request to the same endpoint
while one request is still pending.  You can also open the app in a browser
on your mobile device and try it there as well.

## License

This code is licensed under Apache v2.  See the LICENSE file in the root of
the repository.

## Dependencies

For a list of 3rd party dependencies that are used see the package.json file
in the root of the repository.
