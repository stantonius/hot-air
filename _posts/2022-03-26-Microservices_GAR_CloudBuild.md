---
title: "Creating a Microservice Package Repo with Google Artifact Registry and Cloud Build"
description: "How do you reuse code in multiple Python microservices without copying it?"
layout: post
toc: true
comments: true
hide: false
categories: [GCP, Cloud Build, Artifact Registry, Microservices]
---

# Creating a Microservice Package Repo with Google Artifact Registry and Cloud Build

## Quick Guide to Python Package Management for Microservices with Google Artifact Registry
Straight to what will help people the most - code with as little text as possible. Longer discussion below.

* Assumes:
	*  You have generated a `wheel` file for package distribution
	* Cloud Build used for CI/CD
	* Authenticated to correct GCP project

### 1. Create Repository
```
gcloud artifacts repositories create REPOSITORY_NAME --repository-format=python --location=REGION --description="Repo description"
```

To make your life easier, set defaults for repository and region:

```
gcloud config set artifacts/repository REPOSITORY_NAME
gcloud config set artifacts/location REGION
```

### 2. Authenticate with Google Artifact Registry

Required if **not running in a GCP-managed container service** like Cloud Build (ie. developing and testing *locally*).

#### Standard Authentication
As per the [offical docs](https://cloud.google.com/artifact-registry/docs/python/store-python), you must update `.pypirc` and `pip.conf` to utilize **keyring** authentication. Let GCP tell you what to include by entering the following in the command line:

```
# if you have set defaults for repository, region, and project
gcloud artifacts print-settings python

# if defaults not set
gcloud artifacts print-settings python --project=PROJECT \
--repository=REPOSITORY --location=LOCATION
```

If you've never uploaded a package to Pypi or similar before, you probably won't have either `.pypirc` or `pip.conf` on your machine, so you must create them.

Developing on a Windows machine using a virtual environment, this is where I created them (note you can create globally or locally as well)
```
C:\Users\user\.pypic
C:\Users\user\.virtualenvs\VIRTUALENV_NAME\pip.conf
```

If you do not already have `twine` and the GCP library for keychain validation, install both via the command below:

```
pip install twine keyrings.google-artifactregistry-auth
```

The offical docs recommend you run `keyring --list-backends` to check keyring works. They have a screenshot of what the output looks like. **Don't be concerned with the order of the output** - it might be that the order matters (and why the Mac didn't work), but unless you're well versed in keychain then you can ignore this.

### 3. Upload the Package to the Repository
In the package directory where the `/dist` subdirectory is (that contains the wheel), run the `twine` command below:

```
twine upload --repository-url https://REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME/ dist/*
```

Note that Artifact Registry does not allow package uploads of the same version, so if you get a `HTTPError: 400 Bad Request from https://REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME Bad Request`, you must either create a new package version *or* delete the current package via the console (there is probably a `gcloud` command to do that as well)

If this command returns the error  `EOFError: EOF when reading a line`, GCP does not recognize this as an authenticated request. Thus you may need the alternative authentication as described in the section below.

#### Alternative Authentication (using M1 Mac)
Create a service account that exclusively is for accessing Artifacts (I assigned a newly created service account the role `Artifact Registry Reader`). Download its keys to a json file.

Unlike the above scenario, we need to print the repository configuration by including the *path to the service account credentials*. If you had already done this and had authentication issues, you will need to do again.

```
gcloud artifacts print-settings python --project=PROJECT \
--repository=REPOSITORY --location=LOCATION \
--json-key=KEY-FILE
```

For Mac, here is where I created them (I created these files at the user level, not within a virtual env)
```
$HOME/.pypirc
$HOME/.config/pip/pip.conf
```

Note that I had to *create* the `/pip` directory as well in `.config/`

Now to upload the package to Artifact Registry, you must include your credentials as **base64-encoded** in the request.

> How do you Base64-encode your credentials in a json file? You literally encode the **whole file**. In UNIX you do this via  `cat key.json | base64`

```
twine upload --repository-url https://_json_key_base64:XXXXXXXXXX@REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME dist/*
```

Notice the `_json_key_base64` username and the `:` and `@` separators, and also the credentials `XXXXXX` will be very long

If you are ever prompted for a username and password following this command, just hit Enter. If the credentials worked, you will still be asked for a username and password, but you leave them blank.

#### Another cheeky Authentication Method
Thanks to [this great post] by Lukas Karlsson, there is also another *undocumented* authentication method which uses the `gcloud auth print-access-token` command to get an OAuth2 access token. With this method, the upload URL becomes:

```
twine upload --repository-url https://oauth2accesstoken:XXXXXXXXX@REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME dist/*
```

Where `XXXXXX` is the output of the command `gcloud auth print-access-token`

One thing I noticed about this method was you **don't need any of the keyring/pip.conf/.pypic setup**. You only need `twine`.

### 4. Install a Package hosted in Artifact Registry
This comes back to the original purpose of all of this work - installing a private package multiple times into each of our microservices. The CI/CD tool used is Google's Cloud Build.

There are two ways to install an Artifact Package.

#### Actively Providing Credentials
We can use the same credential methods we just discussed, only this time to download. You will see that nothing changes in the URLs except for the addition of  `/simple` to the endpoint 

Using a service account:
```
pip install --index-url https://https://_json_key_base64:XXXXXXXXX@REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME/simple/ PACKAGE_NAME
```

Using the OAuth2 method:
```
pip install --index-url https://https://oauth2accesstoken:XXXXXXXXX@REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME/simple/ PACKAGE_NAME
```

##### Building with Docker locally
If you want to use a Docker container *and test locally*, you will need to set the `GOOGLE_APPLICATION_CREDENTIALS` for the Docker container (thanks to Kenji Koshikawa's [post](https://dev.to/koshilife/manage-private-python-packages-using-artifact-registry-google-cloud-30kh) that made this very clear). In Kenji's article, you can see in the `Dockerfile` config that they set this environment variable to the path to the service account credentials *on the local machine* (remember: the credentials are only needed **during the build** to install the package; they're not needed to run the container after setup)

```Dockerfile
ENV GOOGLE_APPLICATION_CREDENTIALS "key.json"

# ...

RUN pip install keyrings.google-artifactregistry-auth
RUN pip install --no-cache-dir -r requirements.txt
```

##### `requirements.txt`
The syntax for installing a package from Artifact Registry using `requirements.txt` is: 

```
--extra-index-url https://REGION-python.pkg.dev/PROJECT_ID/REPOSITORY_NAME/simple
sample-repository==0.0.1
```

#### Using Cloud Build Default Credentials
Cloud Build tells us that its service account credentials [are automatically authorized](https://cloud.google.com/functions/docs/writing/specifying-dependencies-python#private_dependencies_from_artifact_registry). In order to take advantage of this, you need to first run the **Python Cloud Builder** and use `pip` to install our private repo from Artifact Registry **into `/workspace`**. The gist of how this was set up is [here](https://gist.github.com/stantonius/957e8b336b714d8db63480fb3d1a6480).

If for some reason you need to use another persistent volume outside of `/workspace`, you would need to add an intermediary step to the Build to *copy* the installed package into `/workspace`. An example of this `cloudbuild.yaml` can be found [here](https://gist.github.com/stantonius/0de6d0c32ffbcd818b233b28762fd84e).

Initially I had tried to `pip install` the package artifact *as part of the Dockerfile*. This was a huge time sink until I realized that the **Docker Builder container does not have access to the Cloud Build service account credentials**. What this meant was if in the `Dockerfile` we tried to `pip install` our private package, we would get the same `EOFError: EOF when reading a line` error, telling us we needed to authenticate some way.

Also note that the directory name **`/workspace` is not accessible inside the Dockerfile**. Instead, the *current directory is `/workspace`* and is accessible via `.`

```Dockerfile
# ...
WORKDIR /code
COPY /workspace/installs /code       # /workspace not accessible
COPY ./installs /code                # this will work - '.' is /workspace
```


##### Using Private Packages
You just saw we installed private packages into a non-standard directory. In order to use them, you need to point the `PYTHONPATH` environment variable to the directory that contains these packages.

```
ENV PYTHONPATH="/code/installs" 
```

For anyone who is unaware what `PYTHONPATH` does (like I was), it is another place that Python will look for packages. It is evaluated *before* the Python interpreter runs so that all packages and modules can be imported at any point when running Python.

The complete `Dockerfile` is below:

```
FROM python:3.9-slim-bullseye

WORKDIR /code

ENV PYTHONUNBUFFERED 1

ENV PYTHONPATH="/code/installs"

COPY . /code/

RUN pip install --upgrade -r /code/requirements.txt

CMD ["uvicorn", "main:api", "--host", "0.0.0.0", "--port", "8080"]
```

Because we installed the package into the `/workspace/`, the `COPY` step brings it into the image and thus we set the `PYTHONPATH` to the directory

That is it. The package is now accessible and usable.

## Discussion
A more thorough walkthrough of the observations and rationale.

### Background
I am building a product using GCP-hosted services. The initial design consists of 3 **microservices**: a Slack chatbot, a processing engine, and a messaging platform (to message the user back)

![](https://storage.googleapis.com/afore-public-assets/images/sample_architecture.svg)

What I didn't realize was one significant drawback of a microservice architecture - Don't Repeat Yourself (DRY) problems. It seems obvious now - these microservice environments are completely isolated. If each of the services interact with the same database and use the same SQLAlchemy models, surely we would end up with functions and classes that *could* be used in all of the containers. 

A quick search revealed that this is a very common dilemma and is a known drawback of microservice architecture. A summarization of the solutions to this problem are below:
1. DO repeat yourself - copy code to each of the containers (either manually or via a script or third-party service)
2. Abstracting the repeating code as a microservice itself. This was not ideal since what would need to be sent between services was tokens, which I wanted to avoid (despite being behind a VPC). Furthermore, this code I wanted in each of the containers was a mixture of functions and classes, not really designed as a standalone service.
3. Creating a private Python package that can be installed on each of the services (just like any other package)


### GCP Artifact Registry

The Google Artifact Registry (GAR) stores the private Python package. You may have heard of Google Container Registry before - GAR is the evolution of that service, allowing storage and access of not only containers, but also code packages as well.

I decided to use this GCP service thinking it made sense given the integration of this service with tools I was using regularly: Cloud Build and Run.

> Difference between a Python module, library, package can be found [here](https://learnpython.com/blog/python-modules-packages-libraries-frameworks/)

Why not Github or Pypi?

Fair question - both of those solutions would also work and might even be less of a headache. However I a) wanted to try GAR and b) using this service means one less set of credentials to manage (or so I thought\*) since Cloud Build and Cloud Run are automatically authenticated for the Artifact , but GAR allows for everything to be managed by GCP (and because I am using Cloud Build and Cloud Run, there is no additional authentication that is needed)

>  \* This is true if you are running or building containers *within* the GCP services Cloud Build and Cloud Run. However, developing locally is a different story

There are other useful features of GAR that are beyond the scope of this post as they benefit big teams (ie. the ability to restrict access to different repos based on GCP user roles)

### Artifact Registry Authentication

This, plain and simple, was an unpleasant experience. This keyring approach that is recommended in [the docs](https://cloud.google.com/artifact-registry/docs/python/store-python#setup-venv) just never worked for me on the M1 Mac. And even using a Windows machine, it seemed like a lot of boilerplate (maybe this is standard for artifact/package management?). All I can say is this seemed like a lot more work than needed (and what it was worth, given ultimately I could have got the same features using a private Github repo). 

Developing locally or in a Docker container required constant authentication or transfering of credentials to the image. The only way you can really take advantage of the Cloud Build [automatic authorization](https://cloud.google.com/functions/docs/writing/specifying-dependencies-python#private_dependencies_from_artifact_registry) is if you Cloud Build emulators locally, which I really dislike using because the responsiveness of these containers is so slow compared to local development using virtual environments or Docker containers.


## Summary
Highlights:
* Microservices that share code elements can use a package repo as one alternative to copy/pasting code in different locations; packages for repeat installation can be stored in Github, Pypi, or Google Artifact Registry
* Storing a private Python package in Google Artifact Registry is difficult and not user friendly. However from the time spent on this, it does look like other artifact registries *may* also be similar.
* The solution to develop locally depended on the machine used: with the M1 Mac, a dedicated service account and password authentication was the only solution that worked for me; on Windows I was able to avoid any additional steps other than those in the offical docs

Is Artifact Registry worth the hassle, especially as a one-person band right now? No, probably not, especially given the Githib repo option. However I will probably continue to use it now that I have figured this out, so long as I can use bash scripts to automate all of this credential encoding when developing locally. I am glad however, that my intuition on how to manage microservice code distribution is aligned to how others have approached this common problem.


## Helpful Resources
* Excellent post from [Kenji Koshikawa](https://dev.to/koshilife/manage-private-python-packages-using-artifact-registry-google-cloud-30kh) who also took this Artifact Python package approach to solve the microservice code copy problem
* A Cloud Build Github Trigger solution by [Lukas Karlsson](https://lukwam.medium.com/python-packages-in-artifact-registry-d2f63643d2b7). They have some cool scripts and also a helpful undocumented trick on how to authenticate in an easier way