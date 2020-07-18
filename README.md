Jupyter is a tool for running interactive notebooks; basically add Python with Markdown and you've got Jupyter. if you haven't used it before, I recommend you do. 

In this post, I'm going to show you how to deploy a Jupyter Notebook server on Heroku using Docker. 

## The big caveat
Jupyter has the ability to create new notebooks and they will 100% save on your deployed docker-based Jupyter server... but they will **disappear** as soon as you deploy a new version. That's because containers, by their very nature, are ephemeral by default. 

This caveat doesn't mean we shouldn't do this... it just means it is a HUGE consideration when using this guide over something like http://colab.research.google.com.

Near the bottom, I'll show you how to package all your Jupyter contents, download it, and unpackage it again when you deploy.


### Final Project Structure

```
cfe_jupyter
|   Dockerfile
│   Pipfile  
│   Pipfile.lock
│
└───conf
│   │   jupyter.py
|
└───nbs
│   │   notebook.tar.gz
│   
└───scripts
    │   Dockerfile
    │   d_build.sh
    |   d_run.sh
    |   deploy.sh
    |   entrypoint.sh
```

## How it's done.

#### 1. Use `pipenv` and install `jupyter`

```
pip install pipenv
cd path/to/your/project/
pipenv install jupyter --python 3.8
```

#### 2. Create Jupyter Configuration

**Generate Default Config**
```
jupyter notebook --generate-config
```
This command creates the default `jupyter_notebook_config.py` file on your local machine. Mine was stored on `~/.jupyter/jupyter_notebook_config.py`

**Create `conf/jupyter.py`**
```
mkdir conf
echo "" > conf/jupyter.py
```
In `conf/jupyter.py` add:

```python
import os
c = get_config()
# Kernel config
c.IPKernelApp.pylab = 'inline'  # if you want plotting support always in your notebook
# Notebook config
c.NotebookApp.notebook_dir = 'nbs'
c.NotebookApp.allow_origin = u'cfe-jupyter.herokuapp.com' # put your public IP Address here
c.NotebookApp.ip = '*'
c.NotebookApp.allow_remote_access = True
c.NotebookApp.open_browser = False
# ipython -c "from notebook.auth import passwd; passwd()"
c.NotebookApp.password = u'sha1:8da45965a489:86884d5b174e2f64e900edd129b5ef0d2f784a65'
c.NotebookApp.port = int(os.environ.get("PORT", 8888))
c.NotebookApp.allow_root = True
c.NotebookApp.allow_password_change = True
c.ConfigurableHTTPProxy.command = ['configurable-http-proxy', '--redirect-port', '80']
```
A few noteable setup items here:

- `c.NotebookApp.notebook_dir` I set as `nbs` which means you should create a directory as `nbs` for your default notebooks directory. In my case, jupyter will open right to this directory ignoring all others.
- `c.NotebookApp.password` - this has to be a hashed password. To create a new one, just run `ipython -c "from notebook.auth import passwd; passwd()"` on your command line.
- `c.NotebookApp.port` - Heroku sets this value in our environment variables thus `int(os.environ.get("PORT", 8888))` as our default.


Test your new configuration locally with: `jupyter notebook --config=./conf/jupyter.py`


#### 3.Create a notebook under -> `nbs/Load_Unload.ipynb`
This will be how you can handle the ephemeral nature of Docker containers with Jupyter notebooks. Just create a new notebook called `Load_Unload.ipynb`, and add the following:

```python
mode = "unload"

if mode == 'unload':
    # Zip all files in the current directory
    !tar chvfz notebook.tar.gz *

elif mode == 'load:
    # Unzip all files in the current directory
    !!tar -xv -f notebook.tar.gz
```


#### 4. Add your `Dockerfile`
This is the absolute minimum setup here. You might want to add additional items as needed. Certain packages, especially the ones for data science, require additional installs for our docker-based linux server.

```dockerfile
FROM python:3.8.2-slim

ENV APP_HOME /app
WORKDIR ${APP_HOME}

COPY . ./

RUN pip install pip pipenv --upgrade
RUN pipenv install --skip-lock --system --dev

CMD ["./scripts/entrypoint.sh"]
```
> The most noteable part of this all is that (1) I'm using `pipenv` locally and in docker and (2) I both install `pipenv` and run `pipenv install --system` to install all pipenv dependancies to the entire docker container (instead of in a virtual environment within the container as well).



#### 5. Create `scripts/entrypoint.sh`

I perfer using a `entrypoint.sh` script for the `CMD` in Dockerfiles. 

```bash
#!/bin/bash

/usr/local/bin/jupyter notebook --config=./conf/jupyter.py
```



#### 6. Build & Run Docker Locally

```
docker build -t cfe-jupyter -f Dockerfile .

docker run --env PORT=8888 -it -p 8888:8888 cfe-jupyter
```

#### 7. Heroku Setup

##### 1. Create heroku app
```
heroku create cfe-jupyter
```
- Change `cfe-jupyter` to your app name

##### 2. Login to Heroku Container Registry
```
heroku container:login

```

#### 7. Push & Release To Heroku

```bash
heroku container:push web
heroku container:release web 
```

- `web` is the default for our `Dockerfile`. 
- On the commands above, you might have to append `-a <your-app-name>` like `heroku container:push web -a cfe-jupyter 


#### 8. That's it
```
heroku open
```
This should allow you to open up your project.


## Full Reference



### `Dockerfile`

```dockerfile
FROM python:3.8.2-slim

ENV APP_HOME /app
WORKDIR ${APP_HOME}

COPY . ./

RUN pip install pip pipenv --upgrade
RUN pipenv install --skip-lock --system --dev

CMD ["./scripts/entrypoint.sh"]
```

### `Pipfile`
```
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
jupyter = "*"

[requires]
python_version = "3.8"
```

### `scripts/d_build.sh`
```bash
docker build -t cfe-jupyter -f Dockerfile .
```


### `scripts/d_run.sh`
```bash
docker run --env PORT=8888 -it -p 8888:8888 cfe-jupyter
```


### `scripts/deploy.sh`
```bash
heroku container:push web
heroku container:release web
```


### `scripts/entrypoint.sh`

```bash
#!/bin/bash

/usr/local/bin/jupyter notebook --config=./conf/jupyter.py
```


### `conf/jupyter.py`
```python
import os
c = get_config()
# Kernel config
c.IPKernelApp.pylab = 'inline'  # if you want plotting support always in your notebook
# Notebook config
c.NotebookApp.notebook_dir = 'nbs'
c.NotebookApp.allow_origin = u'cfe-jupyter.herokuapp.com' # put your public IP Address here
c.NotebookApp.ip = '*'
c.NotebookApp.allow_remote_access = True
c.NotebookApp.open_browser = False
# ipython -c "from notebook.auth import passwd; passwd()"
c.NotebookApp.password = u'sha1:8da45965a489:86884d5b174e2f64e900edd129b5ef0d2f784a65'
c.NotebookApp.port = int(os.environ.get("PORT", 8888))
c.NotebookApp.allow_root = True
c.NotebookApp.allow_password_change = True
c.ConfigurableHTTPProxy.command = ['configurable-http-proxy', '--redirect-port', '80']
```


### Create a notebook under -> `nbs/Load_Unload.ipynb`

```python
mode = "unload"

if mode == 'unload':
    # Zip all files in the current directory
    !tar chvfz notebook.tar.gz *

elif mode == 'load:
    # Unzip all files in the current directory
    !!tar -xv -f notebook.tar.gz
```