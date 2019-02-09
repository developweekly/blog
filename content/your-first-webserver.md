---
title: "Your First Python Tornado Webserver"
date: 2019-02-09T12:05:22+03:00
---

Through the on-boarding process of the internship in my company, we get a lot of questions from juniors. Since our company is a startup and our team is relatively small, we haven't prioritised creating an on-boarding documentation yet. The biggest question mark is always how the web works. What is a web server? On this blog post, which is going to be the first of a series, I will show how to set up a simple web server that handles a pair of HTTP requests in Python.

`Tornado` is the Python module we are going to use. In order to install Python packages, you need to have Python installed - which you will have out of the box if you're using Linux - and a tool for installing Python packages (ie, pip or pipenv).

I will be using Ubuntu and I encourage you to obtain any Linux distribution of your choice as well if you intend to code in Python. To install pip package manager:


```bash
sudo apt-get install -y python-pip
```

And using pip, you can now install Tornado:


```bash
pip install tornado
```

Now let's follow Tornado's `Hello, World` guide and create a simple application by modifying it a little bit. You can check the original sample application from: [http://www.tornadoweb.org/en/stable/#hello-world](http://www.tornadoweb.org/en/stable/#hello-world)

Let's create a file and name it `webserver.py` with your favorite text editor (I prefer VSCode on macOS and emacs on terminal) and copy the lines below in it:


```python
import tornado.ioloop
import tornado.web

class HelloHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

    def post(self):
        user = self.get_argument("username")
        passwd = self.get_argument("password")
        self.write("Your username is {user} and password is {passwd}".format(user=user, passwd=passwd))

def make_app():
    return tornado.web.Application([
        (r"/", HelloHandler),
    ])

if __name__ == "__main__":
    app = make_app()
    app.listen(8080)
    tornado.ioloop.IOLoop.current().start()
```

Here, on line with `def make_app():`, we are defining the routes of our application. In our application, there's only one route, which is the origin `/`. This is the route you are directed when you go to any address like `somesite.com` without any further path. And we map our origin path to a handler called `HelloHandler`.

On line with `class HelloHandler`, we have defined a handler called `HelloHandler`, which is a class inheriting Tornado's `tornado.web.RequestHandler` class. In our handler, we define `get` and `post` methods. These methods will be called when an HTTP `GET` or `POST` request is made to our web server. In `get` method, we simply respond to the request with a friendly message, `Hello, world`. In `post` method we are expecting two arguments namely `username` and `password`. These are just arbitrary arguments and can be anything you like or nothing at all. And we respond to the request with the values of the arguments we receive.

Finally, we are wrapping our web server and will be running it on port `8080` which is defined as `app.listen(8080)`. To run the web server, just save this file and then run this in shell:


```bash
python webserver.py
```

Now, your web server is listening to HTTP requests on address `localhost:8080`, try opening this url in your browser and you will get the welcoming message.

Similarly, you will get a message for the POST requests you make such as:


```python
import requests

args = {"username": "myuser", "password": "mypass"}
response = requests.post("http://localhost:8080", data=args)
print(response.text)
>>> 'Your username is myuser and password is mypass'
```

You can't send a POST request using browser directly afaik unless you are using Firefox. But it's possible to do so using Postman, or Javascript.
