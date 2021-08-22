## An example of a simple flask application

### In this post

- Python 3.7+

____

### Background

I needed to build a very simple one-page flask application, to be access from a domain.

The requirement be that it's low cost and needed to host some static information and accessible via a web address.

I was able to host it on a domain provided by the client, I just needed to re-point the name servers.

It included an `ini` file, which I have at the bottom also (to create one, paste the contents into notepad, and save it as "example.ini" (with the quotation marks) into the same folder

### How to view results

On your machine, navigate to 127.0.0.1 in your web browser of choice.

It has been defaulted to port 80, but you can change the port in the very last line to 5000 instead, which traditionally is a test port.


## Flask Application

```python
import logging
from flask import Flask, redirect
from configparser import ConfigParser

log = logging.getLogger('werkzeug')
log.setLevel(logging.ERROR)

#=======================================================================================


# Start the application in global scope, on load
app = Flask(__name__)

# If you need a configuration file, you can read it from here
config = ConfigParser()
config.read(r'example.ini')



# Found on hanif-a.github.io



#=======================================================================================



@app.route("/")
def index():

    # app.route / - this is the landing page
    # app.route /text-here is another page e.g. www.example.com/text-here

    # I want to take a value from a configuration file in the same directory
    display_name = config["Page Info"]["example_name"]

    # Add page contents here in HTML, to return
    page_html =  f'''
		<html> 

		<h1>Hello {display_name}.</h1>        

		</html>
	'''

    return page_html



@app.route('/text-here')
def code_to_run_on_this_page():

    # Don't do anything, just redirect when they go to /text-here
    pass

    # Send a redirect back
    return redirect('/')



#=======================================================================================


if __name__ == '__main__':
	
	app.run(debug=False, host="0.0.0.0", port=80)
```

.ini file to include in the same folder before opening

```
[Page Info]
example_name=Foo Bar
```
