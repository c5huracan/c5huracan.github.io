# Demonstrating a FastHTML Web Application on Google Colab

## Introduction

FastHTML represents a powerful new approach to web application development, combining the simplicity of HTML templates with the flexibility of Python and the performance of modern web frameworks. Built on top of Starlette, FastHTML makes it easy to create interactive web applications with minimal boilerplate code.

Google Colab offers an attractive platform for developing and sharing web applications, especially for data scientists and developers who want to showcase their work without managing complex infrastructure. With its free GPU/TPU access, pre-installed libraries, and collaborative features, Colab provides an excellent environment for experimentation and demonstration.

However, running web servers in Colab presents unique challenges. Colab notebooks are designed primarily for data analysis and machine learning tasks, not for hosting web applications. The ephemeral nature of Colab instances, networking limitations, and the inability to directly expose ports to the internet create obstacles that require creative solutions.

In this tutorial, we'll walk through the process of deploying a FastHTML search evaluation application on Google Colab. We'll address common challenges including:

- Configuring a web server to accept external connections in Colab's environment
- Setting up proper file paths and dependencies
- Creating and managing search indices for the application
- Exposing your application to the internet using ngrok
- Troubleshooting common issues that arise in this unique hosting environment

By the end of this guide, you'll have a fully functional web application running in your Colab notebook, accessible from anywhere with an internet connection. This approach enables you to quickly prototype and share interactive applications without the need for traditional web hosting services.

Let's begin by setting up our environment and understanding the components of our FastHTML application.

## Prerequisites

Before diving into our Colab deployment, let's ensure you have everything needed to follow along with this tutorial. The process is designed to be accessible, but a few fundamentals will make your experience smoother.

### Google Colab Account

You'll need access to Google Colab, which is free with a Google account. If you're new to Colab, it's essentially a Jupyter notebook environment that runs in the cloud, allowing you to write and execute Python code in your browser. Simply visit [colab.research.google.com](https://colab.research.google.com) and sign in with your Google account to get started.

### Basic Python Knowledge

While this tutorial doesn't require advanced programming skills, familiarity with Python basics will help you understand what's happening in each step. You should be comfortable with:
- Running Python code in notebooks
- Understanding basic syntax and function calls
- Working with files and directories
- Installing packages using pip

### The FastHTML Application Code

For this tutorial, we'll be working with a search evaluation tool built with FastHTML. This application allows users to search through content, evaluate results, and analyze the effectiveness of different search algorithms. You'll need access to the repository containing this code.

If you're following along with your own FastHTML application, you can adapt these instructions to fit your specific needs. The core principles remain the same regardless of the particular application you're deploying.

### Required Dependencies

Our application relies on several Python packages:
- `fasthtml`: The core framework for our web application
- `monsterui`: A UI component library that works with FastHTML
- `lancedb`: A vector database for storing and searching embeddings
- `sentence-transformers`: For creating text embeddings
- `bm25s`: For keyword-based search functionality
- `rerankers`: For improving search result rankings
- `pyngrok`: For exposing our local application to the internet

Don't worry about installing these packages just yet—we'll cover that in the next section. I've listed them here so you understand the components that make our application work.

With these prerequisites in mind, let's move on to setting up our environment in Google Colab.

## Setting Up the Environment

Now that we understand what we need, let's prepare our Google Colab environment for running the FastHTML application. This process is straightforward and requires just a few steps to get everything working properly.

### Creating a New Colab Notebook

Start by opening a new Colab notebook:
1. Go to [colab.research.google.com](https://colab.research.google.com)
2. Select "New Notebook" to create a fresh workspace
3. Rename your notebook (optional) by clicking on "Untitled" at the top of the page

### Cloning the Repository

First, we'll clone the specific repository containing our FastHTML search application:

```python
!git clone https://github.com/Isaac-Flath/search-starter-demo.git
```

This downloads all the necessary files into a directory named `search-starter-demo` within your Colab environment.

### Installing Required Packages

Instead of installing packages individually, we can use the requirements file included in the repository:

```python
!pip install -r ./search-starter-demo/requirements.txt
```

This will install all the necessary dependencies for the application in one go.

### Fixing File Paths

One common issue when running applications in Colab is that file paths may not match what the application expects. We need to modify the path in the search index creation script:

```python
# Read the file content
with open('./search-starter-demo/create_search_index.py', 'r') as file:
    content = file.read()

# Replace the path
modified_content = content.replace("Path('rendered_posts/markdown')", "Path('./search-starter-demo/rendered_posts/markdown')")

# Write back to the file
with open('./search-starter-demo/create_search_index.py', 'w') as file:
    file.write(modified_content)
```

This ensures that the script will look for the markdown files in the correct location within our Colab environment.

### Creating Search Indices

Now we can run the script to build the search indices:

```python
!python3 ./search-starter-demo/create_search_index.py
```

This process may take a moment as it reads the content, creates embeddings, and builds the necessary search indices.

With our environment set up and indices created, we're ready to move on to preparing the application to run in Colab's unique environment.

## Preparing the Application for Colab

Once we have our environment set up and search indices created, we need to make a few adjustments to the application to ensure it works properly in Google Colab. The main challenge is that Colab doesn't allow direct access to the application running on its servers—we need to explicitly configure it for external access.

### Understanding Localhost Limitations

By default, web applications in Python bind to `localhost` or `127.0.0.1`, which means they only accept connections from the same machine. In a local development environment, this works fine since you're accessing the application from the same computer it's running on.

However, in Google Colab, the application runs on Google's servers, not your local machine. If the application binds only to localhost, you won't be able to access it from your browser. We need to modify the application to bind to `0.0.0.0` (all available network interfaces) and set up proper port forwarding.

### Modifying the Server Configuration

Let's modify the `main.py` file to change how the server starts:

```python
# In a Colab cell:
with open('./search-starter-demo/main.py', 'r') as f:
    code = f.read()

# Replace the serve() line
modified_code = code.replace('serve()', 'serve(host="0.0.0.0", port=5001)')

with open('./search-starter-demo/main.py', 'w') as f:
    f.write(modified_code)
```

This change tells the FastHTML server to:
1. Listen on all network interfaces (`0.0.0.0`), not just localhost
2. Use port 5001, which we'll expose through ngrok in the next step

### Setting Up ngrok for External Access

Even with our application configured to accept external connections, we still need a way to access it from outside Google's network. This is where ngrok comes in—it creates a secure tunnel from the public internet to our application running in Colab.

First, let's install the pyngrok package:

```python
!pip install pyngrok
```

Next, we need to set up ngrok with an authentication token. If you don't already have an ngrok account, you'll need to:
1. Sign up at [ngrok.com](https://ngrok.com)
2. Get your auth token from the ngrok dashboard
3. Store it securely in Google Colab's secrets manager

To set your ngrok token and create a tunnel:

```python
from pyngrok import ngrok
import os

from google.colab import userdata
ngrok.set_auth_token(userdata.get('ngrok'))

# Start a tunnel on port 5001
http_tunnel = ngrok.connect(5001)
print(f"Public URL: {http_tunnel.public_url}")
```

This code:
1. Retrieves your ngrok auth token from Colab's secure storage
2. Sets up ngrok with your authentication token
3. Creates a tunnel to port 5001 on the Colab instance
4. Prints the public URL you can use to access your application

The URL provided will look something like `https://a1b2c3d4.ngrok.io` and will remain active as long as your Colab session is running.

With these preparations complete, we're now ready to start our application and access it through the ngrok URL.

## Running the Application

With our environment configured and ngrok tunnel established, we're now ready to launch the FastHTML application. This is the moment of truth where we'll see our search evaluation tool come to life in the cloud environment.

### Starting the Server

Starting the server is straightforward now that we've made the necessary modifications. Simply run the following command in a new Colab cell:

```python
!python3 search-starter-demo/main.py
```

When you execute this command, you'll notice that the cell remains in a "running" state and displays output from the server. This is normal and expected—the server process is actively running and handling requests.

You'll see output similar to this:

```
INFO:     Uvicorn running on http://0.0.0.0:5001 (Press CTRL+C to quit)
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

This confirms that the server is running and listening for connections on all network interfaces (0.0.0.0) on port 5001.

### Accessing Your Application

Now, you can access your application through the ngrok URL that was printed earlier when you set up the tunnel. It should look something like `https://a1b2c3d4.ngrok.io`.

Open this URL in a new browser tab, and you should see the FastHTML search evaluation interface. The application is now running on Google's servers but accessible to you (and anyone else you share the URL with) from anywhere with internet access.

### Understanding the Server Process

It's important to note a few things about how the server runs in Colab:

1. **Blocking execution**: The cell running the server will continue to execute and won't allow you to run additional code in that cell.

2. **Output logs**: The cell running the server will display logs and error messages, which can be helpful for debugging.

3. **Session limitations**: The application will only run as long as your Colab session remains active. If you close your browser or if Colab disconnects due to inactivity (typically after 90 minutes of no interaction), your server will stop.

### Interacting with the Application

You can now use the search evaluation tool as if it were running on a traditional web server:

1. Enter search queries in the search box
2. Select different search methods (Vector, BM25, Hybrid, or Rerank)
3. View and evaluate search results
4. Save your evaluations to the database

All of these interactions will be processed by the FastHTML application running in your Colab environment, with data stored in the database files within your Colab session.

### Stopping the Server

If you need to stop the server to make changes or restart it, you have a few options:

1. **Interrupt the kernel**: Click the "Stop" button (square icon) next to the cell running the server.

2. **Restart the runtime**: Click "Runtime" in the top menu, then "Restart runtime".

3. **Programmatically**: In a new cell, run:
   ```python
   import os
   import signal
   os.kill(os.getpid(), signal.SIGINT)
   ```

With the application up and running, let's move on to troubleshooting common issues that might arise during this process.

## Troubleshooting Common Issues

Even with careful setup, you might encounter challenges when running a FastHTML application in Google Colab. Let's explore some common issues and their solutions to help you navigate potential roadblocks.

### Database and File Path Problems

**Issue**: "Table 'blog_chunks' was not found" or similar database errors.

**Solution**: This typically happens when the search indices haven't been properly created. Ensure you've:
1. Run the `create_search_index.py` script successfully
2. Modified the file paths correctly to point to the right location
3. Checked for any error messages during index creation

If problems persist, try examining the database file:
```python
!ls -la *.db
```

### Connection and Access Issues

**Issue**: "403 Forbidden" errors when trying to access your application.

**Solution**: This could be due to:
1. CORS (Cross-Origin Resource Sharing) restrictions
2. ngrok authentication issues
3. Server configuration problems

Try adding CORS middleware to your application or ensure your ngrok authentication is set up correctly. You might also need to check if your server is actually binding to `0.0.0.0` by examining the server logs.

### Server Process Management

**Issue**: The server crashes or stops unexpectedly.

**Solution**:
1. Check the error logs for specific issues
2. Ensure you have sufficient memory in your Colab instance
3. Consider upgrading to Colab Pro for more resources if you're running complex applications

For unexpected terminations, implement a monitoring approach:
```python
import time
import subprocess

def monitor_server(process):
    while True:
        if process.poll() is not None:
            print("Server process terminated. Restarting...")
            process = subprocess.Popen(["python3", "search-starter-demo/main.py"],
                                      stdout=subprocess.PIPE,
                                      stderr=subprocess.STDOUT)
        time.sleep(10)  # Check every 10 seconds
```

### Dependency and Version Conflicts

**Issue**: Errors related to incompatible package versions.

**Solution**:
1. Check for specific version requirements in the error messages
2. Install specific versions of packages:
   ```python
   !pip install package-name==specific.version
   ```
3. Consider creating a requirements file with pinned versions for reproducibility

### Colab Session Timeouts

**Issue**: Colab disconnects after periods of inactivity, terminating your server.

**Solution**:
1. Use a simple keepalive script to prevent timeouts:
   ```python
   %%javascript
   function ClickConnect(){
     console.log("Clicking connect button"); 
     document.querySelector("colab-connect-button").click()
   }
   setInterval(ClickConnect, 60000)
   ```
2. Upgrade to Colab Pro for longer running sessions
3. Save your work frequently and implement a quick restart procedure

### Browser Cache Issues

**Issue**: Old versions of your application appear even after making changes.

**Solution**:
1. Use hard refresh in your browser (Ctrl+F5 or Cmd+Shift+R)
2. Clear browser cache and cookies
3. Try accessing the application in an incognito/private window

By anticipating these common issues and knowing how to address them, you can save valuable time and maintain a smoother development experience when working with web applications in Google Colab.

## Best Practices

Running a web application in Google Colab can be a powerful way to demonstrate your work, but it requires some thoughtful approaches to ensure stability, security, and performance. Let's explore some best practices to help you get the most out of this setup.

### Maintaining Your Colab Session

Colab sessions will disconnect after periods of inactivity, which can be frustrating when running a server. To maximize uptime:

1. **Implement session keepalive techniques**:
   Beyond the JavaScript approach mentioned earlier, consider interacting with your notebook regularly or using libraries like `colab_keepalive` that can help maintain your connection.

2. **Use efficient resource management**:
   Colab has memory limits that, when exceeded, can cause your session to crash. Monitor your memory usage and clean up unnecessary resources:
   ```python
   # Check memory usage
   !free -h
   
   # Clear unused variables
   import gc
   gc.collect()
   ```

3. **Create quick restart procedures**:
   Document the exact steps needed to restart your application, or better yet, create a dedicated cell with all commands needed to restart your server after a disconnection.

### Security Considerations

When exposing a web application through ngrok, you're making it accessible on the public internet, which introduces security concerns:

1. **Limit exposure time**:
   Only keep your ngrok tunnel open when you need it. Close it when you're done:
   ```python
   ngrok.disconnect(http_tunnel.public_url)
   ```

2. **Use authentication**:
   Consider adding basic authentication to your FastHTML application or use ngrok's authentication features (available in paid plans).

3. **Be cautious with sensitive data**:
   Avoid storing sensitive information in your application when running it on Colab with public access. If you need to demonstrate with real data, consider using anonymized or synthetic datasets.

4. **Monitor access logs**:
   Check who's accessing your application by examining the server logs or ngrok's inspection interface.

### Performance Optimization

Colab has limited resources, so optimizing your application's performance is important:

1. **Minimize dependencies**:
   Only install the packages you absolutely need. Each additional package adds overhead.

2. **Use efficient search algorithms**:
   If your application involves search (like our example), choose algorithms that balance accuracy and performance based on your dataset size.

3. **Implement caching**:
   Cache computation-heavy results to avoid redundant processing:
   ```python
   from functools import lru_cache
   
   @lru_cache(maxsize=100)
   def expensive_computation(param):
       # Your resource-intensive code here
       return result
   ```

4. **Load models efficiently**:
   If using machine learning models (like sentence transformers in our example), load them only once and reuse the instances.

### Reproducibility and Documentation

To ensure you can quickly recreate your environment:

1. **Version your requirements**:
   Maintain an up-to-date requirements file with specific versions:
   ```python
   !pip freeze > requirements.txt
   ```

2. **Document your setup**:
   Create a README or notebook cells that clearly explain the setup process, including any manual steps.

3. **Use environment variables for configuration**:
   Store configuration in environment variables rather than hardcoding:
   ```python
   import os
   from google.colab import userdata
   
   # Get configuration from Colab secrets
   API_KEY = userdata.get('api_key')
   ```

4. **Create setup scripts**:
   Automate as much of the setup process as possible with scripts to minimize manual intervention.

By following these best practices, you'll create a more robust, secure, and performant web application deployment in Google Colab, making it a more reliable platform for demonstrations and prototyping.
## Conclusion

Running a FastHTML web application on Google Colab represents an innovative approach to web development that leverages cloud resources in unconventional ways. While Colab wasn't designed specifically for hosting web applications, it offers a surprisingly effective platform for prototyping, demonstrations, and educational purposes.

Throughout this tutorial, we've explored how to transform Colab from a data science notebook environment into a temporary web hosting solution. We've addressed the unique challenges of this approach—from localhost limitations to session timeouts—and provided practical solutions for each obstacle.

The search evaluation application we've deployed showcases the potential of this setup. It demonstrates how complex functionality like vector search, BM25 keyword retrieval, and reranking can all be made accessible through a browser interface, without dedicated hosting infrastructure.

However, it's important to recognize the limitations of this approach. Colab's ephemeral nature means your application won't persist indefinitely. The resource constraints and timeout policies make it unsuitable for production applications or situations requiring high reliability.

For more permanent or production-ready solutions, consider platforms like:
- Railway (I hear good things from people in the know)
- Render, Fly.io, or PythonAnywhere for their free tiers
- Streamlit Cloud for data-focused applications
- Traditional cloud providers like AWS, GCP, or Azure for scalable solutions

That said, Colab provides an excellent starting point for:
- Quickly prototyping web interfaces
- Sharing interactive demonstrations
- Teaching web development concepts
- Testing application functionality before committing to a hosting platform

By combining FastHTML's simplicity, Colab's accessibility, and ngrok's connectivity, you've created a powerful development environment that costs nothing to run. This approach democratizes web application development, allowing anyone with a Google account to experiment with building and sharing interactive web applications.

A special thanks to Isaac Flath for creating the original search-starter-demo repository that served as the foundation for this tutorial. His work in developing a clean, functional example of a FastHTML search application made this guide possible and provided an excellent learning resource for the community.

As you continue your development journey, remember that the skills you've learned here—configuring servers, managing processes, handling file paths, and troubleshooting connectivity issues—are transferable to more traditional hosting environments. The experience gained from working within Colab's constraints will make you a more resourceful and adaptable developer.

Thank you for following along with this tutorial. We hope it empowers you to create, experiment, and share your FastHTML applications with the world, regardless of your infrastructure budget or hosting experience.
